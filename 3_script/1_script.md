We recently had a workshop on domain driven design, where I we really mixed together OOP and DDD. Since in our Project we are thinking about moving towards a more functional architecture, I asked, how the process of modelling that we did in the workshop would work with that. Two claims were made:

1. Domain Driven Design is necessarily object oriented
2. Other approaches are inherently less efficient to maintain and extend.

I think I know where those ideas come from. I think a lot of early DDD Books are very Java/DDD centric, so that may be why.

I want to go into these two claims and work out my differences with them.

# Claim 1: DDD is inherently object oriented.

DDD is about modelling business concepts, creating ubiquitous language, defining bounded contexts.

## What is DDD 
### Core Concepts

**Ubiquitous Language**

**Bounded Contexts**

**Domain Model**

### Building Blocks

**Entities**

**Value Objects**

**Aggregates**

**Repositories**

**Domain Services**

**Domain Events**

### Strategic Design

- Focus on the most valuable/complex parts (Core Domain)
- Integration patterns between bounded contexts
- Context mapping to visualize relationships


## Does DDD work if implemented functionally

The Domain should be modelled based on a common understanding of Language. Intuitively, you might think that only works using Objects, since that is how we commonly model the World. 

Example:

```typescript
class Customer {
  constructor(private readonly id: string, private name: string) {}

  createOrder(items: OrderItem[]): Order {
    return new Order(this.id, items);
  }
}

class Order {
  constructor(
    public readonly customerId: string,
    private items: OrderItem[]
  ) {}
}

class OrderItem {
  constructor(private productId: string, private quantity: number) {}
}
```

The ubiquitous language "Customer creates Order" maps directly to customer.createOrder(). That is consistent with this interpretation of the world.

![Object Oriented Modelling](modelling_context.png)

But, we can just as easily model the world as a set of processes of cause and effect (data transformations).

![Compare](compare.png)


## How functional DDD might look

**Ubiquitous Language**
In OOP: class names, method names, property names in ubiquitous language

In FP: instead of naming classes, name types and functions
Instead of Order.approve(), you might have a function approveOrder : Order -> Order.

**Bounded Contexts**
In OOP: Mapped to module, package, namespace containing related classes.

In FP: Mapped to module, or namespace related to types and functions.
There is no shared mutable state accross boundaries, since functional code has explicit dependencies and pure composition.

**Domain Model**
In OOP: hierarchies where behavior and state is coupled


In FP: 
a set of pure types and functions that model the real world.

**Entities**
In OOP:
Entities are objects with identity and mutable state.

In FP: 
Entities are immutable data structures that contain their identity and any state at a point in time.

**Value Objects**
In OOP:
Often small, immutable classes with value equality.

In FP: simple types, equality is derived automatically. One step further is pattern matching in functional languages (like elixir).

**Repositories**
In OOP: classes that hide persistence mechanics behind oop interfaces.

In FP: functions that represent persistence behavior that can be injected as dependencies.

**Aggregates**
In OOP:
Objects that encapsulate entities to enforce correct mutation

In FP:
Aggregates are functions that take current state + command → new state (+ events). That's all thats needed to enforce invariants, since there's no mutation.

**Domain Services**
In OOP:
Classes encapsulating domain logic that doesn’t belong to a single entity.


In FP:
Pure functions grouped in a module
If side effects are needed, they’re made explicit by returning them as data (e.g., commands or events), so they don't do those directly.

**Domain Events**
In OOP:
Events raised by entities or aggregates that something meaningful happened

In FP:
Immutable Data Records

## Claim 2: Functional Programming is inherently more complex.

The claim was, functional programming falls into the category transaction script, and therefore, it is harder to maintain.

### What is a transaction script vs domain model


![Martin Fowler's domain logic patterns](graph.jpg)


I had to first look into transaction scripts, to understand what  they are.

> Transaction Script is a pattern where you organize business logic as a series of procedures, with each procedure handling a single request/transaction from the presentation layer.

**Example of transaction script**

```typescript
class OrderService {
  async placeOrder(userId: string, items: CartItem[]) {
    if (!items.length) throw new Error("Cart is empty");
    
    // All logic in one place
    let total = 0;
    for (const item of items) {
      total += item.price * item.quantity;
    }
    
    await db.createOrder(userId, total);
  }
}
```

**Example of domain model**

```typescript
class Order {
  private items: OrderItem[];
  private status: OrderStatus;
  
  calculateTotal(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.subtotal()),
      Money.zero()
    );
  }
  
  cancel(): void {
    if (!this.status.canTransitionTo(OrderStatus.Cancelled)) {
      throw new Error("Cannot cancel order in current state");
    }
    this.status = OrderStatus.Cancelled;
  }
}

// Service is thin orchestration
class OrderService {
  async placeOrder(userId: string, items: CartItem[]): Promise<string> {
    const order = new Order(items); // Domain logic in object
    return await this.orderRepo.save(order);
  }
}
```

**Differences between transaction script and domain model**

| Aspect | Transaction Script | Domain Model |
|--------|-------------------|--------------|
| **Logic location** | In service procedures | In domain objects |
| **Reusability** | Copy-paste between procedures | Shared methods on objects |
| **Complexity** | Linear growth | Scales better |
| **Encapsulation** | Exposed data | Protected invariants |



I agree, domain model is probably easier to maintain. And transaction script is probably what most of our application is doing. It's easy to see why you would think, functional programming can only be transaction script though.


### Why the confusion
If you are used to thinking in classes, not having a class to you feels like there is no model. In functional programming, the domain model emerges from the shape of the data, the collection of functions that define how that data is transformed and by module boundaries.

```typescript
class Order {
  private items: OrderItem[];
  
  calculateTotal(): Money { /* logic here */ }
  cancel(): void { /* logic here */ }
}
```

vs

```typescript
type Order = { items: OrderItem[] };

const calculateTotal = (order: Order): Money => /* logic */;
const cancel = (order: Order): Result<Order, Error> => /* logic */;
```


### What really are the differences and would those differences result in different complexities?

**Encapsulation**
In OOP: Data is encapsulated within the object.

In FP: Data is not encapsulated within objects, but you still control how the domain is interacted with by creating a module interface and choosing what is exported vs. private.

Seeming OOP advantage: As a developer, it appears harder to create side effects when data is encapsulated. The interface may be easier to visualize.

Reality: You have to define the same interface in both paradigms. Actually, in FP, data is treated as immutable within functions, so you have fewer possible side effects. 

**Structuring the Project**
In OOP: To find data and logic, you look in the class.

In FP DDD: To find data and logic, you look in the module. Everything is in one namespace.

**Understanding state changes**
In OOP: Aggregates must track how mutations can happen through carefully controlled methods.

In FP: Everything is immutable. The old state still exists. State changes are explicit through function composition, making the evolution of state easier to trace. You can still have those same 'control' methods.

### Example from earlier done functionally
An implementation of the earlier example of domain model code, done functionally, looks like:

```typescript
type Order = {
  readonly items: readonly OrderItem[];
  readonly status: OrderStatus;
};

type OrderStatus = 
  | { type: 'Pending' }
  | { type: 'Cancelled', reason: string };

// Domain logic as functions operating on immutable data
const calculateTotal = (order: Order): Money =>
  order.items.reduce(
    (sum, item) => addMoney(sum, subtotal(item)),
    zeroMoney()
  );

const cancel = (reason: string) => (order: Order): Result<Order, Error> => {
  if (order.status.type !== 'Pending') {
    return Err(new Error('Cannot cancel non-pending order'));
  }
  return Ok({
    ...order,
    status: { type: 'Cancelled', reason }
  });
};

// Application service remains thin
const placeOrder = async (
  userId: UserId,
  items: readonly CartItem[]
): Promise<OrderId> => {
  const order = createOrder(items); // Domain function
  return await orderRepo.save(order);
};
```