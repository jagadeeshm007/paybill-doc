# Engineering the PayBill Forms: Validation Binding and the Approval Engine

Forms are inherently difficult in mobile development. Combine complex forms with asynchronous financial approval workflows, and you have a recipe for brittle, hard-to-maintain code.

In PayBill, we completely rethought how data flows from user input to final financial approval. This post dives into our Form Validation Binding and the proprietary Approval Flow Engine.

---

## 1. Form Validation Binding: Zod + React Hook Form

In traditional React Native apps, forms are often uncontrolled arrays of `useState` hooks with custom validation logic stacked at the end. This scales terribly.

For PayBill, we enforce **strict schema-driven design**.

### The Flow

1. **Define the Schema:** Using Zod, we define the exact shape the API expects. This includes Regex for UPI IDs, number limits for amounts, and string length requirements.
2. **Bind the Form:** We use `React Hook Form` tied tightly to the Zod resolver.
3. **Instant Feedback:** As the user types, standard React Hook Form handlers evaluate the Zod schema instantly without blocking the UI thread. Errors are mapped directly to corresponding fields.

This pattern completely divorces our UI components from business logic. A text input component doesn't know _why_ an amount is invalid, it just knows the validation engine has flagged it, allowing it to turn red and display the provided error string.

---

## 2. The Approval Flow Engine

Creating a transaction is just the beginning. The core of PayBill is its **Approval Flow Engine**.

When a user submits a transaction, it doesn't just write to a database; it enters a strict state machine with defined lifecycle stages.

### The Happy Path

1. **Submitted:** Transaction is logged. Awaiting administrator review.
2. **Approved:** Administrator approves. Awaiting financial admin review.
3. **Financial Approved:** Finance approves. Ready for banking systems.
4. **Uploaded to Bank:** Sent to bank for processing.
5. **Payment Done, Awaiting Bills:** Payment complete, waiting for the sender to attach physical receipts.
6. **Bills Accepted:** Accountant verifies physical bills. Process complete.

### Designing the State Machine

We handle this complexity using an event-driven architecture.

When a transaction is opened in the app (`app/transaction/[id].tsx`), the client reads the current status string (e.g., `Payment Done, Awaiting Bills`). Based on the active user's role (see our RBAC architecture), the `Approval Engine` calculates which actions they are legally allowed to take.

For instance, if the status is `Approved`, a `user` sees a read-only timeline. A `financialadmin` sees an "Approve" and "Deny" button.

### The Approval Timeline Component

To make this transparent to the end-user, we built the `ApprovalTimeline` component. This visualizes every state change, documenting _who_ made the decision and _when_. It's a localized Git-blame for financial transactions, ensuring absolute accountability.

---

## 3. Handling Denials and Edge Cases

Not every transaction succeeds. The engine handles complex denial paths:

- **Admin Denied:** Kicked back to the user to edit and resubmit.
- **Bills Quality Failed:** The accountant rejects the upload, requiring the user to take a clearer picture of the receipt.

By strictly mapping these states and relying on our Zod schemas to ensure data integrity at every step of resubmission, the PayBill Approval Engine creates a frictionless, auditable, and immutable record of every penny processed through the system.
