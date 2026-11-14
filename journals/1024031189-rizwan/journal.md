# UCS503P Engineering Journal — Rizwan Handa (1024031189)

* **Project Title:** IndoorPulse — Infrastructure-Free Multi-Floor Indoor Positioning & 3D Topological Routing System
* **Student Name:** Rizwan Handa
* **Roll Number:** 1024031189
* **GitHub Handle:** `@rizwanhanda`
* **Repositories:** `rizwanhanda/IndoorPulse-Android` | `rizwanhanda/navix`
* **Primary Test Hardware:** Nothing Phone (2) (Qualcomm Snapdragon 8+ Gen 1, 12GB LPDDR5 RAM, 512GB UFS 3.1)

---

## Week 1: Introduction and Pleasantries
**Focus:** Course onboarding and preliminary domain analysis.
**Notes:** Attended the initial laboratory sessions and finalized the group formation with Harjot, Ravinder, and Pooja. We conducted a preliminary review of potential project domains, analyzing the technical feasibility of enterprise web platforms versus mobile sensor applications. We agreed to pursue a high-impact engineering challenge that requires deep hardware-software integration rather than a standard CRUD application.

## Week 2: Group Formation & Practise Project Pitch
**Focus:** Evaluating indoor localization architectures and hardware constraints.
**Notes:** Investigated the feasibility of building a campus navigation application. Initial research focused on Wi-Fi RSSI (Received Signal Strength Indicator) fingerprinting and Bluetooth Low Energy (BLE) beacon trilateration. However, our feasibility study concluded that these methodologies require prohibitive capital expenditure for hardware deployment, suffer from environmental fragility when router configurations change, and experience catastrophic failure during power outages. Consequently, we pivoted to researching a 100% infrastructure-free approach using smartphone Inertial Measurement Unit (IMU) sensors.

## Week 3: Elevator Pitch
**Focus:** Finalizing the infrastructure-free dead-reckoning concept.
**Notes:** Pitched the core "IndoorPulse" architecture to the instructor: a completely offline indoor navigation system utilizing the built-in accelerometer and gyroscope. The primary feedback highlighted the inherent risk of cumulative sensor drift in dead-reckoning systems. We committed to engineering a probabilistic filtering algorithm to mathematically constrain and correct this drift, ensuring long-term localization stability without external hardware beacons.

## Week 4: Project Proposal & Feasibility Report
**Focus:** Formalizing the technology stack and defining SMART objectives.
**Notes:** Drafted and submitted the official Project Proposal, detailing our architectural pipeline and expected deliverables.
*   **Tech Stack Chosen:** Kotlin, Jetpack Compose, Android `SensorManager`, and Room SQLite.
*   **SMART Goal:** Achieve ≥ 95% step-detection accuracy and sub-2.0m localization drift over a 50-meter trajectory.
*   **MWE (Project Setup):** Configured the foundational Compose UI and Room database Gradle dependencies to establish the project baseline.
```kotlin
// build.gradle.kts (Module: app)
dependencies {
    implementation("androidx.compose.ui:ui:1.5.0")
    implementation("androidx.room:room-runtime:2.6.0")
    kapt("androidx.room:room-compiler:2.6.0")
    implementation("com.google.android.gms:play-services-location:21.0.1")
}
```

## Week 5: Progress Monitoring (Sensor Polling)
**Focus:** Accessing and normalizing IMU hardware streams.
**Notes:** Conducted initial data logging using raw `TYPE_ACCELEROMETER` on the test device. We discovered a critical noise factor: the 9.81 m/s² gravitational acceleration vector introduces significant interference into the horizontal X and Y axes whenever the user naturally alters the pitch or roll of the handset. To resolve this, we migrated the polling pipeline to utilize Android's hardware-fused `TYPE_LINEAR_ACCELERATION` and `TYPE_ROTATION_VECTOR`.
```kotlin
// MWE: IMU Sensor Registration (KinematicPDR.kt)
val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
val linearAccel = sensorManager.getDefaultSensor(Sensor.TYPE_LINEAR_ACCELERATION)
val rotationVec = sensorManager.getDefaultSensor(Sensor.TYPE_ROTATION_VECTOR)

// Polling initialized at 50Hz to optimize the balance between signal resolution and battery consumption
sensorManager.registerListener(this, linearAccel, SensorManager.SENSOR_DELAY_GAME) 
sensorManager.registerListener(this, rotationVec, SensorManager.SENSOR_DELAY_GAME)
```

## Week 6: Progress Monitoring (Schmitt Trigger Implementation)
**Focus:** Engineering a robust Pedestrian Dead Reckoning (PDR) step detector.
**Notes:** Standard zero-crossing algorithms proved inadequate, registering false steps due to stationary oscillation artifacts (e.g., the phone vibrating on a desk). We engineered a Dual-Threshold Schmitt Trigger utilizing hysteresis banding. This approach arms the step cycle only upon crossing a high threshold and registers the step upon crossing a lower threshold, enforced by a 260ms refractory debounce period to aggressively reject hand-jitter.
```kotlin
// MWE: Dual-Threshold Schmitt Trigger Logic
if (!isArmed && accelMag >= 1.85f) {
    isArmed = true
    peak = accelMag
} else if (isArmed && accelMag <= 0.45f && (nowMs - lastStepMs) >= 260L) {
    // Fused with Weinberg biomechanical stride estimation
    val strideMeters = (0.43 * Math.pow((peak - valley).toDouble(), 0.25)).toFloat()
    onStepDetected(strideMeters)
    isArmed = false
    lastStepMs = nowMs
}
```

## Week 7: Prototype (MST Prep)
**Focus:** UI rendering and preliminary field testing.
**Notes:** Submitted the First Prototype. While the PDR engine accurately logged step counts, rendering the trajectory on a Jetpack Compose `Canvas` revealed a coordinate system chirality mismatch. Walking forward and executing a 90-degree physical right turn resulted in a left turn on the screen. This discrepancy occurs because the Android Azimuth frame (North-East-Down) is right-handed, whereas the screen-space Canvas (+Y projecting downward) is left-handed.

## Week 8: Improvement Plan (Addressing Chirality & Drift)
**Focus:** Mathematical corrections for coordinate mapping and drift containment.
**Notes:** Formulated a comprehensive improvement plan to implement a Monte Carlo Particle Filter. This stochastic architecture will serve a dual purpose: providing a structured environment to apply explicit trigonometric frame translations (resolving the chirality bug) and strictly bounding accumulated IMU drift by clamping simulated position estimates to the physical boundaries of walkable corridors.

## Week 9: MST Week
**Focus:** Mid-Semester Examinations.
**Notes:** Suspended active software development to prioritize preparation for theoretical midterm examinations across concurrent courses.

## Week 10: MST Week
**Focus:** Theoretical modeling for the Particle Filter architecture.
**Notes:** Reviewed MST evaluation outcomes. Conducted research on Low-Variance Systematic Resampling algorithms to ensure our particle filter maintains computational efficiency. Drafted the spatial data structures required to project a cloud of 400 discrete particles onto the Room SQLite topological maps without blocking the main UI thread.

## Week 11: Progress Monitoring (Over Improvement Plan)
**Focus:** Deploying the 400-Particle Monte Carlo Localization (MCL) filter.
**Notes:** Successfully integrated the MCL filter. We applied explicit trigonometric translation matrices to correct the Week 7 chirality anomaly. Furthermore, particles that mathematically drift outside the boundaries of the topological corridor map are now heavily penalized, effectively eliminating lateral drift accumulation during straight corridor walks.
```kotlin
// MWE: Particle Propagation & Chirality Translation (ParticleFilter.kt)
for (p in particles) {
    val noisyStride = stride + randomGaussian() * 0.06f
    val noisyHeading = azimuth + randomGaussian() * 0.08f
    
    // Explicit transformation: Clockwise NED Azimuth to +Y Down Screen-Space Canvas
    p.x += noisyStride * kotlin.math.sin(noisyHeading)
    p.y -= noisyStride * kotlin.math.cos(noisyHeading)
}
```

## Week 12: Progress Monitoring (Room SQLite & Surveyor Mode)
**Focus:** Developing the on-device topological graph builder.
**Notes:** Finalized the `Room` relational database persistence layer (`BuildingEntity`, `NodeEntity`, `EdgeEntity`). Developed "Surveyor Mode" within `MapCanvas.kt`, featuring dynamic touch hit-detection. This allows system administrators to physically map nodes and draw interconnecting corridors directly on the handset, bypassing the need for complex external CAD software.
```kotlin
// MWE: Dynamic Touch Snapping for Surveyor Mode (MapCanvas.kt)
fun findTappedNode(touchPx: Offset, scale: Float, pan: Offset): NodeEntity? {
    // Dynamically scale the touch radius to preserve selection accuracy at various zoom levels
    val hitRadiusMeters = (24f / scale).coerceIn(0.4f, 1.8f)
    val worldX = (touchPx.x - pan.x) / scale
    val worldY = (touchPx.y - pan.y) / scale
    return nodes.find { kotlin.math.hypot(it.x - worldX, it.y - worldY) <= hitRadiusMeters }
}
```

## Week 13: Buffer (Cloud Infrastructure Setup)
**Focus:** Architecting a compliant, self-hosted cloud infrastructure.
**Notes:** As course guidelines prohibited the use of managed Platform-as-a-Service (PaaS) providers like Vercel or Firebase, we provisioned an AWS EC2 Linux virtual machine. We deployed a Python Flask REST API to handle authenticated web uploads, configuring it to pipe deliverables and JSON map artifacts directly to a Backblaze B2 S3-compatible storage bucket.
```python
# MWE: Authenticated Flask to Backblaze B2 S3 Upload Route
@app.route('/api/upload', methods=['POST'])
def upload():
    if request.form.get('password') != ADMIN_SECRET_KEY:
        return jsonify({'error': 'Unauthorized'}), 403
    file = request.files['file']
    b2_client.upload_fileobj(file, BUCKET_NAME, file.filename)
    return jsonify({'success': True, 'url': f"{B2_ENDPOINT}/{file.filename}"})
```

## Week 14: Progress Monitoring (3D A* Routing)
**Focus:** Implementing multi-floor pathfinding algorithms.
**Notes:** Expanding our topological graphs to encompass multiple floors exceeded the capabilities of our initial 2D Dijkstra pathfinder, resulting in invalid cross-floor routing artifacts. We upgraded the routing engine to a 3D Euclidean A* heuristic, applying a substantial vertical weight penalty to force cross-floor transitions strictly through designated `vertical_portal` stairwell edges.
```kotlin
// MWE: 3D Euclidean A* Heuristic (MapViewModel.kt)
private fun heuristic3D(a: NodeEntity, b: NodeEntity): Float {
    val dx = a.x - b.x
    val dy = a.y - b.y
    // Impose a 4.0m average floor height penalty to optimize vertical traversal logic
    val dz = (a.floorLevel - b.floorLevel) * 4.0f 
    return kotlin.math.sqrt(dx * dx + dy * dy + dz * dz)
}
```

## Week 15: Second Prototype (JSON Engine & Transit State)
**Focus:** Ensuring data portability and mitigating stairwell drift.
**Notes:** Developed a comprehensive JSON Export/Import engine utilizing UUID translation to prevent SQLite primary key collisions when synchronizing maps across multiple devices. During field testing, we observed that traversing stairs induces severe cumulative errors in horizontal displacement calculations. To counter this, we engineered a proximity-based "Transit State" state machine that automatically suspends step integration when the user's estimated position falls within 1.5m of a stairwell portal.

## Week 16: Progress Monitoring (Web Console Integration)
**Focus:** Translating the 3D A* simulation engine to a browser environment.
**Notes:** Developed the `navix` HTML5 Canvas interface. We resolved a defect where CSS flexbox scaling misaligned mouse click hitboxes by applying internal resolution multipliers. Furthermore, we refactored the simulation loop to utilize `performance.now()` time-deltas, ensuring the simulation runs deterministically regardless of whether the client monitor operates at 60Hz or 100Hz.
```javascript
// MWE: Device-Agnostic DPI-Scaled Canvas Coordinates (index.html)
function getCanvasMouseCoords(e) {
    const rect = mapCanvas.getBoundingClientRect();
    return {
        // Multiplier ensures exact internal pixel targeting regardless of CSS viewport stretching
        x: (e.clientX - rect.left) * (mapCanvas.width / rect.width),
        y: (e.clientY - rect.top) * (mapCanvas.height / rect.height)
    };
}
```

## Week 17: Final Prototype and Presentation
**Focus:** Interface refinement, Emergency Egress execution, and final deployment.
**Notes:** Finalized the aesthetic and structural elements of the Web Map Console. Rigorously validated the Emergency Egress (SOS) routing logic, confirming that the 3D pathfinder strictly filters and routes toward `Building Entry` ground-level nodes rather than arbitrarily terminating at indoor waypoints. Compiled the final academic project report in LaTeX, verified the AWS EC2 cloud gateway connectivity, and deployed the complete technology stack. The system is verified, stable, and fully operational.
