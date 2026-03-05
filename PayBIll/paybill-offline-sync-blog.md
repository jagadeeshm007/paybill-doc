<div align="center">
  <p><i>PayBill Engineering Series &middot; Chapter 5</i></p>
  <a href="./README.md">← Back to Table of Contents</a>
</div>

---

# The Edge: Auto Re-Sync and Edge Reliability Architecture

Mobile devices lose signal. They transition from 5G to spotty edge networks, drop connections in elevators, and disconnect entirely on subways.

An enterprise payment application like PayBill cannot simply throw a "Network Error. Try again later" alert and delete the user's heavily validated form data. Our users require an application that fails gracefully and recovers automatically.

This post explores the Offline/Online architecture and Auto Re-Sync capabilities engineered into PayBill.

---

## 1. The Offline Data Problem

In a standard REST architecture, a user fills out a transaction form and hits "Submit". The app POSTs to the server. If the network drops during the POST, what happens?

If the app doesn't handle this state locally:

1. The transaction is fully lost.
2. The UI freezes.
3. The user accidentally submits duplicates when mashing the "Try Again" button.

### Core Architecture Shift

To solve this, PayBill utilizes a local cache layer utilizing robust tools from the React Native ecosystem, notably `@react-native-async-storage/async-storage` and heavily customized Zustand stores (`use-sync-external-store`).

---

## 2. Resilient Transaction Drafts

To handle offline-first flows and user-interrupted actions gracefully, we implemented resilient **Transaction Drafts**.

If a user begins a payment but loses signal (or simply needs to stop to retrieve a physical receipt), they can manually or automatically save the transaction as a "Draft".

### How Drafts Work:

1. **Local Writes:** The transaction metadata (vendor, amount, project ID, timestamp) is serialized and stored directly onto the device's persistent `AsyncStorage`.
2. **Metadata Versioning:** We attach unique offline IDs to these drafts so they can be securely tracked.
3. **Session Resumption:** When the user returns to the app, the "Saved Drafts" tab reads the local SQLite/AsyncStorage cache. The user can resume exactly where they left off, even miles away from a stable internet connection.

![The Transactions List view showing the separation between synced history and persistent Local Drafts.](./assets/images/transactions-list.png)

---

## 3. The Auto Re-Sync Engine

Drafts solve intentional pausing, but what about unintentional disconnection during submission?

We built a specialized Auto Re-Sync flow:

1. **The Sync Queue:** When a transaction submission fails specifically due to network timeout or offline status, the parsed Zod-validated payload is pushed into an encrypted local sync queue rather than discarded.
2. **Status Identification:** The transaction is optimistically added to the user's history with an `Offline Pending` visual tag.
3. **The Listener:** We utilize native network state listeners to monitor the device's connection status.
4. **Silent Processing:** As soon as an active data connection is verified, a background process (`async-limiter` managed queue) picks up the pending transactions and attempts to flush them to the Supabase backend sequentially.
5. **Reconciliation:** Once the server confirms the transaction and returns the true database ID, the local history updates silently, removing the `Offline Pending` tag.

---

## 4. Conflict Resolution and Edge Cases

The most complex part of offline syncing is conflict resolution. What if an Admin denied a transaction while the user's device was offline, but the user attempted to edit and resubmit it while still offline?

Our Approval Engine handles this by strictly versioning timestamps array parameters. The Supabase backend operates on a "Last Server Write Wins" policy with strict constraints. If the backend detects a status mismatch between the client's offline payload and the server's current reality, it rejects the flush attempt and triggers a UI notification to the user to manually reconcile the difference.

By combining persistent local caching, a background sync queue, and strict server-side state enforcement, PayBill guarantees extreme reliability regardless of the user's physical environment.

---

<div align="center">
  <a href="./README.md">← Return to Table of Contents</a>
</div>
