---
title: "On Consul ACL Radix Tree"
date: 2023-08-22T13:20:11+10:00
draft: false
---

I recently spent time looking at Consul ACL sub-system in depth. And these are my findings.

Consul ACL system secures access to the UI, API, CLI, service-to-service and agent-to-agent communication.

The system works by distributing artifact to users, called ACL token. When users make request to Consul, the token will be transformed to list of associated rules and Consul will enforce the rules to determine if the token has access to requested resources.

I created this graph to make sense on the token attributes.

{{< mermaid >}}
flowchart TB
    token --> policy
    policy --> rule
    role --> policy
    role --> service-identity
    role --> node-identity
    service-identity --> policy
    node-identity --> policy
    token --> service-identity
    token --> node-identity
    token --> role
{{< /mermaid >}}


# How an ACL token get resolved to list of rules?

The token passed to Consul server will go through this process to get transformed into an `Authorizer` struct. 

{{< mermaid >}}
flowchart LR
    subgraph token-resolve
        token-->identity
        identity-->policies
        policies-->single-policy
        single-policy-->cache
        single-policy-->rules
        rules-->id1{authorizer}
        cache-->id1
    end
    subgraph policy-enforcement
    end
    subgraph acl
        token-resolve --> policy-enforcement
    end
{{< /mermaid >}}

We can refer to the code to validate above flow.

Starts with method `ResolveTokenAndDefaultMeta` being called

```go
	// Fetch the ACL token, if any.
	authz, err := c.srv.ResolveTokenAndDefaultMeta(args.Token, &args.EnterpriseMeta, nil)
	if err != nil {
		return err
	}
```

The token get resolved into identity and policies

```go
	identity, policies, err := r.resolveTokenToIdentityAndPolicies(tokenSecretID)
```

Detail

```go
	for i := 0; i < tokenPolicyResolutionMaxRetries; i++ {
		// Resolve the token to an ACLIdentity
		identity, err := r.resolveIdentityFromToken(token)
		if err != nil {
			return nil, nil, err
		} else if identity == nil {
			return nil, nil, acl.ErrNotFound
		} else if identity.IsExpired(time.Now()) {
			return nil, nil, acl.ErrNotFound
		}

		lastIdentity = identity

		policies, err := r.resolvePoliciesForIdentity(identity)
```

All policies will be de-duped

```go
	// Now deduplicate any policies or service identities that occur more than once.
	policyIDs = dedupeStringSlice(policyIDs)
	serviceIdentities = serviceIdentities.Deduplicate()
	nodeIdentities = nodeIdentities.Deduplicate()

	// Generate synthetic policies for all service identities in effect.
	syntheticPolicies := r.synthesizePoliciesForServiceIdentities(serviceIdentities, identity.EnterpriseMetadata())
	syntheticPolicies = append(syntheticPolicies, r.synthesizePoliciesForNodeIdentities(nodeIdentities, identity.EnterpriseMetadata())...)

	// For the new ACLs policy replication is mandatory for correct operation on servers. Therefore
	// we only attempt to resolve policies locally
	policies, err := r.collectPoliciesForIdentity(identity, policyIDs, len(syntheticPolicies))
	if err != nil {
		return nil, err
	}

	policies = append(policies, syntheticPolicies...)
```

And compiled into a single policy

```go
	authz, err := policies.Compile(r.cache, &conf)
```

The compile process will check if there's cache for the policy and use it

```go
func (policies ACLPolicies) Compile(cache *ACLCaches, entConf *acl.Config) (acl.Authorizer, error) {
	// Determine the cache key
	cacheKey := policies.HashKey()
	entry := cache.GetAuthorizer(cacheKey)
	if entry != nil {
		// the hash key takes into account the policy contents. There is no reason to expire this cache or check its age.
		return entry.Authorizer, nil
	}
```

Else it will go ahead, merge all policies into one and create an `Authorizer` struct that load all rules associated with the policy.

```go
func newPolicyAuthorizerFromRules(rules *PolicyRules, ent *Config) (*policyAuthorizer, error) {
	p := &policyAuthorizer{
		agentRules:         radix.New(),
		intentionRules:     radix.New(),
		keyRules:           radix.New(),
		nodeRules:          radix.New(),
		serviceRules:       radix.New(),
		sessionRules:       radix.New(),
		eventRules:         radix.New(),
		preparedQueryRules: radix.New(),
	}

	p.enterprisePolicyAuthorizer.init(ent)

	if err := p.loadRules(rules); err != nil {
		return nil, err
	}
```

AHA! 🤩 This is where the Radix Tree is created, each of the ACL `resource`, e.g. `Service`, `Node`, `KV` will have 1 Radix tree created to support the policy lookup later.

With this shiny `Authorizer` struct created, the ACL check will be a lookup on the right Radix tree to see if there's any exact/prefix rule for the resource.

For example, this is the ACL check when there's a query to read a `Service`

```go
			// Check ACLs.
			authz, err := s.agent.delegate.ResolveTokenAndDefaultMeta(token, nil, nil)
			if err != nil {
				return "", nil, err
			}
			var authzContext acl.AuthorizerContext
			svc.FillAuthzContext(&authzContext)
			if err := authz.ToAllowAuthorizer().ServiceReadAllowed(svc.Service, &authzContext); err != nil {
				return "", nil, err
			}
```

This will call `ServiceRead` method on the `Authorizer`

```go
// ServiceRead checks if reading (discovery) of a service is allowed
func (p *policyAuthorizer) ServiceRead(name string, ctx *AuthorizerContext) EnforcementDecision {
	// When reading a service imported from a peer we consider it to be allowed when:
	//  - The request comes from a locally authenticated service, meaning that it
	//    has service:write permissions on some name.
	//  - The requester has permissions to read all services in its local cluster,
	//    therefore it can also read imported services.
	if ctx.PeerOrEmpty() != "" {
		if p.ServiceWriteAny(nil) == Allow {
			return Allow
		}
		return p.ServiceReadAll(nil)
	}
	if rule, ok := getPolicy(name, p.serviceRules); ok {
		return enforce(rule.access, AccessRead)
	}
	return Default
}
```

And the beauty is in the Radix tree lookup of the policy.

```go
// getPolicy first attempts to get an exact match for the segment from the "exact" tree and then falls
// back to getting the policy for the longest prefix from the "prefix" tree
func getPolicy(segment string, tree *radix.Tree) (policy *policyAuthorizerRule, found bool) {
	found = false

	tree.WalkPath(segment, func(path string, leaf interface{}) bool {
		policies := leaf.(*policyAuthorizerRadixLeaf)
		if policies.exact != nil && path == segment {
			found = true
			policy = policies.exact
			return true
		}

		if policies.prefix != nil {
			found = true
			policy = policies.prefix
		}
		return false
	})
	return
}
```

The anonymous function passed to `tree.WalkPath` is a closure with access to `policy` variable, while walking through the Radix tree, if it finds a prefix match, it will store the finding to `policy` variable, but it will keep looking until it hit a exact match and `return true`.

We can refer to the implementation detail of the Radix tree for more info

```go
// WalkPath is used to walk the tree, but only visiting nodes
// from the root down to a given leaf. Where WalkPrefix walks
// all the entries *under* the given prefix, this walks the
// entries *above* the given prefix.
func (t *Tree) WalkPath(path string, fn WalkFn) {
	n := t.root
	search := path
	for {
		// Visit the leaf values if any
		if n.leaf != nil && fn(n.leaf.key, n.leaf.val) {
			return
		}
```

Radix tree is so cool and it is being used across various HashiCorp products such as Consul, Vault and Nomad.