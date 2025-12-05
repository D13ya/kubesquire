# Hour 6: Network Segmentation (Cilium)

**Objective:** Stop data exfiltration (Egress).

This content is located in `walkthroughs/02-identity-segmentation.md`, Section 3.

## Quick Links
- [Open Walkthrough](command:vscode.open?%7B%22resource%22%3A%22c%3A%2FUsers%2Fdibya%2FOneDrive%20-%20Johns%20Hopkins%2FNotes%2FFall%202025%2FCCS%2FProject%2FFinal%2Fwalkthroughs%2F02-identity-segmentation.md%22%7D)

## Tasks
1.  **Lock Down Egress:**
    *   Apply a `CiliumNetworkPolicy` (or standard `NetworkPolicy`) that denies all egress except DNS (53), Google APIs (443), and **Localhost (127.0.0.1)** (required for Istio, see [Issue #18465](https://github.com/cilium/cilium/issues/18465)).
    *   **Test:** Exec into a pod and try `curl google.com`. It should hang/fail.
