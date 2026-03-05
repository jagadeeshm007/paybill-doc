# Building PayBill's Seamless QR Code Scanner and Generator

Mobile payments inherently require interacting with the physical world. In consumer fintech, QR code scanning is the bridge between a physical storefront and a digital transaction.

This post details how PayBill engineered a deeply integrated, always-active QR scanner and a dynamic QR receipt generator.

---

## 1. The Challenge of "Good" Scanning

Most basic React Native QR scanners are frustrating to use. They require users to tap a button, wait for the camera hardware to initialize, point directly at a perfectly lit code, and then manually accept the result.

Our goal for PayBill was different: **Zero friction.**
We wanted an "Always-Active" scanning logic combined with instant deep-linking.

---

## 2. Engineering the "Always-Active" Scanner

We chose not to embed the scanner directly into standard transactional views. Instead, the QR Code scanner was extracted into an isolated feature module: `components/QRImageScanner.tsx`.

### The Implementation Structure

Using Expo Camera APIs, the scanner operates continuously without manual triggering as long as the user is on the target route.

We encountered a significant issue with the device sleeping while scanning ("Unable to activate keep awake" errors). We resolved this by tying the scanner's lifecycle strictly to the standalone route. When the scanner route is active, the camera is active.

### Instant Deep Routing

Upon a successful scan, the string data is instantly parsed. If it matches PayBill's deep link formats (or standard UPI URI formats), Expo Router immediately navigates the user to the transaction creation screen, pre-filling the vendor details and amount. No intermediate "Confirm this code" screens.

We added a distinct haptic impact (`Haptics.notificationAsync`) using native device APIs to provide physical confirmation without requiring the user to look at the screen.

### The Visual Polish

A raw camera feed feels unfinished. We layered our scanner with custom UI rendering:

- **Precision Corner Markers:** Guiding the user's eye to the optimal scanning zone.
- **Continuous Laser Sweep Animation:** Utilizing React Native Reanimated to run a smooth sweeping line over the target area without lagging the primary UI thread.

---

## 3. Dynamic QR Receipt Generation

Scanning is only half the problem. When a PayBill user completes an offline transaction or wants to share their own receiving credentials, they need to _generate_ a code.

### The Share Sheet Overlay

To handle this, we implemented interactive Transaction Drafts and receipts. Upon completing a payment, localized `BottomSheetModal` components generate shareable, dynamic receipts directly within the `app/transaction/[id].tsx` route to avoid complex top-level Z-index navigation conflicts.

These modal sheets incorporate dynamically generated QR codes reflecting the exact transaction hash or the user's receiving URI.

Users can screenshot the generated code or use our "Copy to Clipboard" functionality built directly into the UI overlay.

---

## 4. Conclusion

By separating the QR scanner into a standalone, hardware-accelerated feature route and coupling it with immediate programmatic routing, PayBill eliminates the clunky friction often associated with mobile payments. Our implementation bridges the physical and digital worlds seamlessly, reliably, and instantly.
