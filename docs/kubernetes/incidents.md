# Incident Appendix (Real Scenarios)

## INCIDENT 1: WebSocket / Reverb outage

**Symptoms:**
- Frontend attempts `wss://host:6001`
- 404 or connection refused

**Root Cause:**
- Ingress routed `/app` to app service
- Frontend misconfigured port

**Fix:**
- Route `/app` → `reverb:6001`
- Frontend uses 443

**Prevention:**
- Ingress template enforces per‑path backends
- CI lint for `VITE_REVERB_PORT != 443`

---

## INCIDENT 2: 503 on /login

**Symptoms:**
- Ingress returns 503
- Pods running

**Root Cause:**
- Service name mismatch
- No endpoints

**Fix:**
- Align ingress backend service name

---

## INCIDENT 3: CrashLoopBackOff

**Symptoms:**
- Pod restarts
- Startup probe fails

**Root Cause:**
- Probe too aggressive
- DB unavailable on boot

**Fix:**
- Increase startup delay
- Defer DB‑dependent checks
