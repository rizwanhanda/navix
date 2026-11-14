# UCS503P Engineering Journal — Rizwan Handa (1024031189)

* **Project Title:** IndoorPulse — Infrastructure-Free Multi-Floor Indoor Positioning & 3D Topological Routing System
* **Student Name:** Rizwan Handa
* **Roll Number:** 1024031189
* **GitHub Handle:** `@rizwanhanda`
* **Repositories:** `rizwanhanda/IndoorPulse-Android` | `rizwanhanda/navix`
* **Primary Test Hardware:** Nothing Phone (2) (Qualcomm Snapdragon 8+ Gen 1, 12GB LPDDR5 RAM, 512GB UFS 3.1)

---

## Week 1: Introduction and Pleasantries
**Focus:** Course onboarding and initial brainstorming.
**Notes:** Attended the initial lab sessions and finalized the group formation with Harjot, Ravinder, and Pooja. We discussed general domains of interest, ranging from enterprise web applications to mobile sensor utility apps. Agreed to research high-impact problems that involve hardware-software integration.

## Week 2: Group Formation & Practise Project Pitch
**Focus:** Evaluating project ideas and hardware constraints.
**Notes:** Brainstormed building a campus navigation application. Initially considered utilizing Wi-Fi RSSI fingerprinting or Bluetooth Low Energy (BLE) beacons. However, we realized these approaches require expensive physical hardware deployment, degrade when router configurations change, and completely fail during power outages. Decided to pivot and research if smartphone IMU sensors could handle 100% offline dead-reckoning.

## Week 3: Elevator Pitch
**Focus:** Finalizing the "Infrastructure-Free" concept.
**Notes:** Pitched the core "IndoorPulse" concept to the instructor: a completely offline indoor navigation system using only the smartphone's built-in accelerometer and gyroscope. The feedback highlighted a valid concern regarding cumulative sensor drift. We committed to engineering a probabilistic filter (rather than relying on external beacons) to correct this drift mathematically.

## Week 4: Project Proposal & Feasibility Report
**Focus:** Formalizing the tech stack and SMART objectives.
**Notes:** Drafted and submitted the official Project Proposal. 
*   **Tech Stack Chosen:** Kotlin, Jetpack Compose, Android `SensorManager`.
*   **SMART Goal:** Achieve $\ge 95\%$ step-detection accuracy and sub-$2.0\text{m}$ localization drift.
*   **MWE (Project Setup):** Configured the foundational Compose and Room database Gradle dependencies.
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
**Focus:** Accessing and filtering IMU hardware streams.
**Notes:** Tested raw `TYPE_ACCELEROMETER` on the Nothing Phone (2). Discovered a critical issue: gravity ($9.81\text{ m/s}^2$) bleeds heavily into the horizontal X/Y axes whenever the user naturally tilts the phone in their hand, corrupting movement data. Switched the hardware polling pipeline to the hardware-fused `TYPE_LINEAR_ACCELERATION` and `TYPE_ROTATION_VECTOR`.
```kotlin
// MWE: IMU Sensor Registration (KinematicPDR.kt)
val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
val linearAccel = sensorManager.getDefaultSensor(Sensor.TYPE_LINEAR_ACCELERATION)
val rotationVec = sensorManager.getDefaultSensor(Sensor.TYPE_ROTATION_VECTOR)

// Polling at 50Hz for balance between resolution and battery life
sensorManager.registerListener(this, linearAccel, SensorManager.SENSOR_DELAY_GAME) 
sensorManager.registerListener(this, rotationVec, SensorManager.SENSOR_DELAY_GAME)
```

## Week 6: Progress Monitoring (Schmitt Trigger Implementation)
**Focus:** Building a robust Pedestrian Dead Reckoning (PDR) step detector.
**Notes:** Basic zero-crossing algorithms caused "stationary desk drift"—registering false steps when the phone merely vibrated on a table. Engineered a Dual-Threshold Schmitt Trigger with a $260\text{ms}$ refractory period to aggressively reject hand-jitter, fused with a Weinberg biomechanical stride estimator.
```kotlin
// MWE: Dual-Threshold Schmitt Trigger
if (!isArmed && accelMag >= 1.85f) {
    isArmed = true
    peak = accelMag
} else if (isArmed && accelMag <= 0.45f && (nowMs - lastStepMs) >= 260L) {
    val strideMeters = (0.43 * Math.pow((peak - valley).toDouble(), 0.25)).toFloat()
    onStepDetected(strideMeters)
    isArmed = false
    lastStepMs = nowMs
}
```

## Week 7: Prototype (MST Prep)
**Focus:** Basic UI rendering and initial walk-tests.
**Notes:** Submitted the First Prototype. The PDR engine successfully tracks steps, but rendering them on a Compose `Canvas` revealed a major coordinate chirality bug: walking forward and turning right physically resulted in a left turn on the screen because Android Azimuth (North-East-Down) and Canvas ($+Y$ down) have opposite handedness.

## Week 8: Improvement Plan (Addressing Chirality & Drift)
**Focus:** Mathematical corrections for coordinate mapping.
**Notes:** Devised an improvement plan to implement a Monte Carlo Particle Filter. This architecture will solve the coordinate frame translation (via explicit trigonometric mapping) and strictly bound the accumulated IMU drift by clamping simulated particles to walkable corridor boundaries.

## Week 9: MST Week
**Focus:** Mid-Semester Tests.
**Notes:** Paused active code commits to prepare for theoretical midterm examinations. 

## Week 10: MST Week
**Focus:** Theoretical architecture for the Particle Filter.
**Notes:** Reviewed MST evaluation sheets post-exams. Researched Low-Variance Systematic Resampling algorithms. Sketched out the data structures needed to project $400$ particles onto our upcoming Room SQLite building maps without causing UI thread blockage.

## Week 11: Progress Monitoring (Over Improvement Plan)
**Focus:** Deploying the 400-Particle Monte Carlo Localization (MCL) filter.
**Notes:** Implemented the particle filter and applied explicit trigonometric translation to fix the Week 7 chirality bug. Particles that drift outside the topological corridor bounds are now heavily penalized.
```kotlin
// MWE: Particle Propagation & Chirality Fix (ParticleFilter.kt)
for (p in particles) {
    val noisyStride = stride + randomGaussian() * 0.06f
    val noisyHeading = azimuth + randomGaussian() * 0.08f
    
    // Explicit mapping: Clockwise NED Azimuth to +Y Down Screen-Space Canvas
    p.x += noisyStride * kotlin.math.sin(noisyHeading)
    p.y -= noisyStride * kotlin.math.cos(noisyHeading)
}
```

## Week 12: Progress Monitoring (Room SQLite & Surveyor Mode)
**Focus:** On-device topological graph building.
**Notes:** Built the `Room` relational database persistence layer (`BuildingEntity`, `NodeEntity`, `EdgeEntity`). Implemented Surveyor Mode in `MapCanvas.kt` featuring dynamic touch hit-detection, allowing administrators to map nodes and draw corridors directly on the device.
```kotlin
// MWE: Dynamic Touch Snapping for Surveyor Mode (MapCanvas.kt)
fun findTappedNode(touchPx: Offset, scale: Float, pan: Offset): NodeEntity? {
    // Dynamically scale the touch radius so zooming out doesn't break selection
    val hitRadiusMeters = (24f / scale).coerceIn(0.4f, 1.8f)
    val worldX = (touchPx.x - pan.x) / scale
    val worldY = (touchPx.y - pan.y) / scale
    return nodes.find { kotlin.math.hypot(it.x - worldX, it.y - worldY) <= hitRadiusMeters }
}
```

## Week 13: Buffer (Cloud Infrastructure Setup)
**Focus:** Bypassing PaaS limitations with a custom AWS EC2 backend.
**Notes:** Course requirements prohibited managed services like Vercel or Firebase. Provisioned an AWS EC2 Linux instance running a Python Flask API to handle authenticated web uploads, piping deliverables and JSON map artifacts directly to Backblaze B2 S3 storage.
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
**Focus:** Multi-floor pathfinding logic.
**Notes:** Extending the graph to multiple floors broke our flat Dijkstra router, leading to cross-floor routing bleed. Upgraded to a 3D Euclidean A* heuristic, applying a heavy vertical penalty to restrict cross-floor movement strictly to `vertical_portal` stairwell edges.
```kotlin
// MWE: 3D Euclidean A* Heuristic (MapViewModel.kt)
private fun heuristic3D(a: NodeEntity, b: NodeEntity): Float {
    val dx = a.x - b.x
    val dy = a.y - b.y
    val dz = (a.floorLevel - b.floorLevel) * 4.0f // 4.0m average floor height penalty
    return kotlin.math.sqrt(dx * dx + dy * dy + dz * dz)
}
```

## Week 15: Second Prototype (JSON Engine & Transit State)
**Focus:** Map portability and stairwell drift mitigation.
**Notes:** Built the JSON Export/Import engine with UUID translation to prevent SQLite primary key collisions across devices. Discovered that walking on stairs completely corrupts horizontal PDR (because stacked stairwell nodes share identical $(x, y)$ coordinates). Engineered a proximity-based Transit State modal that automatically pauses step integration within $1.5\text{m}$ of a stairwell portal.

## Week 16: Progress Monitoring (Web Console Integration)
**Focus:** Porting the 3D A* simulator to the browser.
**Notes:** Built the `navix` HTML5 Canvas interface. Fixed a bug where CSS flexbox stretching broke mouse click hitboxes. Refactored the walk simulator to use `performance.now()` time-deltas because frame-counting (`pauseFrames = 50`) ran out of sync on high-refresh-rate $100\text{Hz}$ displays.
```javascript
// MWE: DPI-Scaled Canvas Coordinates (index.html)
function getCanvasMouseCoords(e) {
    const rect = mapCanvas.getBoundingClientRect();
    return {
        // Multiplier ensures exact internal pixel targeting regardless of CSS stretch
        x: (e.clientX - rect.left) * (mapCanvas.width / rect.width),
        y: (e.clientY - rect.top) * (mapCanvas.height / rect.height)
    };
}
```

## Week 17: Final Prototype and Presentation
**Focus:** UI polish, SOS Evacuation Mode, and deployment.
**Notes:** Finalized the Web Map Console styling. Validated the ` SOS Nearest Exit` logic, ensuring the pathfinder strictly filters for `Building Entry` ground-level nodes rather than arbitrary rooms. Compiled the final LaTeX project report, tested the AWS EC2 cloud gateway, and deployed the entire stack to GitHub Pages. System is fully operational and verified.
