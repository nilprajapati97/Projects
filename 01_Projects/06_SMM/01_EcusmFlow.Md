🚦 ECU States (defined by EcuM)
 1. STARTUP: Initial power-on state. Initializes OS, drivers, and BSW.
 2. RUN (UP): Normal operational state. OS and applications running, communication enabled.
 3. SLEEP: Low-power state, hardware in standby, awaiting a wakeup event.
 4. SHUTDOWN: Graceful stop sequence. Applications and modules terminate, hardware powers down.
 5. OFF: Final powered-down state. Exit requires reset or ignition ON.

⸻

🔍 Step-by-Step
 1. Event occurs
 • Examples: Ignition OFF, diagnostic request, network wakeup, ComM timeout.
 2. BSWM Rule Evaluation
 • Example Rule → IF (VSM_IgnitionState == OFF) AND (AllShutdownReasonsMet) THEN RequestShutdown
 • Example Rule → IF (ComM_NoComm) AND (NoWakeupEvent) THEN RequestSleep
 • Example Rule → IF (Ignition ON OR DCM_Active) THEN RequestRUN
 3. BSWM Action
 • Executes corresponding API call:
 • EcuM_RequestShutdown()
 • EcuM_RequestSleep()
 • EcuM_RequestRUN()
 4. EcuM Execution
 • Validates request.
 • Coordinates orderly transitions (asks modules via shutdown targets, handles wakeup validation).
 • Executes detailed sequences:
 • Shutdown: stop OS, de-init drivers, cut power.
 • Sleep: configure hardware, set wakeup sources, enter low-power mode.
 • Run: initialize OS, drivers, enable communications.