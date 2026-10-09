# When to mock

Mocks sit at system boundaries only: external APIs (payment, email), databases (sometimes, with a test database preferred), time and randomness, and sometimes the file system.

Your own classes and modules, internal collaborators and anything you control are never mocked.

## Designing for mockability

At a system boundary, the interface is designed so it is easy to mock.

### Use dependency injection

External dependencies are passed in rather than created internally.

```typescript
// Easy to mock
function processPayment(order, paymentClient) {
  return paymentClient.charge(order.total);
}

// Hard to mock
function processPayment(order) {
  const client = new StripeClient(process.env.STRIPE_KEY);
  return client.charge(order.total);
}
```

### Prefer SDK-style interfaces over generic fetchers

Each external operation gets its own specific function instead of one generic function with conditional logic.

```typescript
// GOOD: Each function is independently mockable
const api = {
  getUser: (id) => fetch(`/users/${id}`),
  getOrders: (userId) => fetch(`/users/${userId}/orders`),
  createOrder: (data) => fetch('/orders', { method: 'POST', body: data }),
};

// BAD: Mocking requires conditional logic inside the mock
const api = {
  fetch: (endpoint, options) => fetch(endpoint, options),
};
```

With the SDK approach each mock returns one specific shape, test setup carries no conditional logic, the endpoints a test exercises are visible at a glance, and each endpoint is typed on its own.
