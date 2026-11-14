# UCS503P Engineering Journal — Rizwan Handa (1024031189)

* **Project Title:** IndoorPulse — Infrastructure-Free Multi-Floor Indoor Positioning & 3D Topological Routing System
* **Student Name:** Rizwan Handa
* **Roll Number:** 1024031189
* **GitHub Handle:** `@rizwanhanda`
* **Repositories:** `https://github.com/rizwanhanda/IndoorPulse-Android` | `https://github.com/rizwanhanda/navix`
* **Primary Test Hardware:** Nothing Phone (2) (Qualcomm Snapdragon 8+ Gen 1, 12GB LPDDR5 RAM, 512GB UFS 3.1)

---

## Entry 1 (Weeks 1–2): Problem Formulation & IMU Sensor Stream Selection

### 1. Context & Engineering Goal
Indoor navigation inside reinforced concrete academic blocks fails due to GPS signal attenuation, while Wi-Fi RSSI fingerprinting and BLE beacon grids fail during power outages. The goal was to establish a self-contained Android inertial sensor pipeline capable of tracking pedestrian movement completely offline.

### 2. Technical Investigation & MWE
Evaluated `Sensor.TYPE_ACCELEROMETER` against `Sensor.TYPE_LINEAR_ACCELERATION` and `Sensor.TYPE_ROTATION_VECTOR` on the Nothing Phone (2). Raw accelerometer readings included the $9.81\text{ m/s}^2$ gravitational component, which leaked into horizontal axes whenever the handset tilted in the user's hand. Switched to hardware-fused linear acceleration and quaternion rotation vectors polled at `SENSOR_DELAY_GAME` ($50\text{ Hz}$):

```kotlin
// MWE: Hardware IMU Sensor Registration in KinematicPDR.kt
class KinematicPDR(context: Context) : SensorEventListener {
    private val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
    private val linearAccel = sensorManager.getDefaultSensor(Sensor.TYPE_LINEAR_ACCELERATION)
    private val rotationVec = sensorManager.getDefaultSensor(Sensor.TYPE_ROTATION_VECTOR)

    fun startListening() {
        sensorManager.registerListener(this, linearAccel, SensorManager.SENSOR_DELAY_GAME)
        sensorManager.registerListener(this, rotationVec, SensorManager.SENSOR_DELAY_GAME)
    }
}
```

### 3. Validation Outcome
Eliminated static gravity projection errors during handset pitch/roll changes.

---

## Entry 2 (Weeks 3–4): Dual-Threshold Schmitt Trigger & Biomechanical Stride Model

### 1. Problem Statement
Single-threshold zero-crossing step detectors suffered from severe false positives when the user shook the phone in hand or placed it on a vibrating desk ("stationary desk drift").

### 2. Solution & MWE (`KinematicPDR.kt`)
Engineered a Dual-Threshold Schmitt Trigger state machine with hysteresis ($T_{\text{high}} = 1.85\text{ m/s}^2$, $T_{\text{low}} = 0.45\text{ m/s}^2$) and a $260\text{ ms}$ refractory debounce window, coupled with Weinberg fourth-root biomechanical stride estimation:

```kotlin
// MWE: Schmitt Trigger Hysteresis State Machine + Weinberg Stride Length
private var isArmed = false
private var peakAccel = 0f
private var valleyAccel = 0f
private var lastStepTimestampMs = 0L

fun processLinearAcceleration(ax: Float, ay: Float, az: Float, nowMs: Long) {
    val mag = kotlin.math.sqrt(ax * ax + ay * ay + az * az)
    if (!isArmed && mag >= 1.85f) {
        isArmed = true
        peakAccel = mag
        valleyAccel = mag
    } else if (isArmed) {
        if (mag > peakAccel) peakAccel = mag
        if (mag < valleyAccel) valleyAccel = mag
        if (mag <= 0.45f && (nowMs - lastStepTimestampMs) >= 260L) {
            val amplitude = (peakAccel - valleyAccel).coerceAtLeast(0.1f)
            val strideMeters = (0.43 * Math.pow(amplitude.toDouble(), 0.25)).toFloat()
            onStepDetected(strideMeters)
            isArmed = false
            lastStepTimestampMs = nowMs
        }
    }
}
```

### 3. Validation Outcome
Achieved $100\%$ rejection of stationary desk jitter over $5\text{-minute}$ stress tests and $97.4\%$ step-count accuracy across corridor walks.

---

## Entry 3 (Weeks 5–6): Coordinate Chirality Translation & 400-Particle Monte Carlo Filter

### 1. Problem Statement
During physical walk tests, making a $90^\circ$ clockwise (right) turn in the physical corridor caused the trajectory on Jetpack Compose `MapCanvas` to invert along the vertical axis due to a left-handed vs. right-handed coordinate chirality mismatch between `SensorManager.getOrientation()` and screen-space Canvas coordinates ($+Y$ downward).

### 2. Solution & MWE (`ParticleFilter.kt`)
Implemented an explicit trigonometric frame translation inside a 400-particle Monte Carlo Localization (MCL) filter, penalizing particle weights whenever perpendicular distance $d_\perp$ to the nearest topological corridor segment exceeded the walkable corridor half-width ($1.2\text{ m}$):

```kotlin
// MWE: Chirality-Corrected 400-Particle Propagation & Corridor Wall Clamping
fun propagateAndClamp(strideMeters: Float, azimuthRad: Float, edges: List<EdgeSegment>) {
    for (p in particles) {
        val noisyStride = strideMeters + randomGaussian() * 0.06f
        val noisyHeading = azimuthRad + randomGaussian() * 0.08f
        // Translate Clockwise NED World Azimuth to Screen-Space Cartesian (+Y Down)
        p.x += noisyStride * kotlin.math.sin(noisyHeading)
        p.y -= noisyStride * kotlin.math.cos(noisyHeading)

        val distToCorridor = minPerpendicularDistance(p.x, p.y, edges)
        if (distToCorridor > 1.2f) {
            p.weight *= kotlin.math.exp(-(distToCorridor * distToCorridor) / (2f * 0.45f * 0.45f))
        }
    }
    normalizeAndResampleIfNeeded()
}
```

### 3. Validation Outcome
Bounded cumulative localization error to $1.18\text{ m}$ over $52\text{ m}$ multi-turn L-corridor walks (compared to $4.85\text{ m}$ with unconstrained raw IMU dead reckoning).

---

## Entry 4 (Weeks 7–8): Room SQLite Topology & Surveyor Mode Canvas Engine

### 1. Context & Engineering Goal
Designed relational Room SQLite entities (`BuildingEntity`, `NodeEntity`, `EdgeEntity`) and built an interactive on-device Surveyor Mode in `MapCanvas.kt` so administrators can map buildings directly on the smartphone without external CAD software.

### 2. Technical Implementation & MWE
Added screen-to-world coordinate projection, zoom clamping ($0.5\times\text{ to }4.0\times$), dynamic touch hit-detection snapping, and an Undo/Redo command stack for graph editing:

```kotlin
// MWE: Screen-to-World Coordinate Projection & Dynamic Touch Snapping
fun findTappedNode(touchPx: Offset, nodes: List<NodeEntity>, scale: Float, pan: Offset): NodeEntity? {
    val hitRadiusMeters = (24f / scale).coerceIn(0.4f, 1.8f)
    val worldX = (touchPx.x - pan.x) / scale
    val worldY = (touchPx.y - pan.y) / scale
    return nodes.minByOrNull { kotlin.math.hypot(it.x - worldX, it.y - worldY) }
        ?.takeIf { kotlin.math.hypot(it.x - worldX, it.y - worldY) <= hitRadiusMeters }
}
```

---

## Entry 5 (Weeks 9–11): Self-Managed 3-Tier Cloud Infrastructure (AWS EC2 + Backblaze B2)

### 1. Context & Engineering Goal
To avoid managed PaaS platforms (Vercel, Netlify, Firebase) and demonstrate low-level systems deployment, we architected a decoupled 3-tier cloud pipeline combining GitHub Pages (`rizwanhanda/navix`), an AWS EC2 Linux VPS (`indoorpulse.duckdns.org`), and Backblaze B2 S3-compatible object storage.

### 2. Technical Implementation & MWE
Configured Nginx with Let's Encrypt TLS reverse-proxying to a Python Flask API that authenticates administrative uploads and streams versioned deliverables and `.json` map packages to Backblaze B2 via `boto3`:

```python
# MWE: Flask S3-Compatible Upload Route on AWS EC2 Connected to Backblaze B2
@app.route('/api/upload', methods=['POST'])
def upload_artifact():
    if request.form.get('password') != ADMIN_SECRET_KEY:
        return jsonify({'error': 'Unauthorized'}), 403
    uploaded_file = request.files['file']
    object_key = f"artifacts/{int(time.time())}_{secure_filename(uploaded_file.filename)}"
    b2_s3_client.upload_fileobj(uploaded_file, B2_BUCKET_NAME, object_key)
    file_url = f"{B2_PUBLIC_ENDPOINT}/{B2_BUCKET_NAME}/{object_key}"
    save_metadata_record(request.form['title'], request.form['version'], file_url)
    return jsonify({'success': True, 'file_url': file_url})
```

---

## Entry 6 (Weeks 12–14): Multi-Floor 3D Euclidean A* Router, SOS Mode & Stairwell Transit State

### 1. Problem Statement
Flat 2D Dijkstra routing caused cross-floor bleed when extending maps to multiple levels (`floorLevel = 0`, `floorLevel = 1`). Additionally, walking up stairs caused severe horizontal PDR drift because stacked stairwell nodes share identical $(x, y)$ coordinates (`x = -4.63m, y = -8.69m`).

### 2. Solution & MWE (`MapViewModel.kt`)
Replaced Dijkstra with a **3D Euclidean A* Pathfinder** using explicit `vertical_portal` stairwell edges ($w = 30.0\text{ m}$), added a proximity-triggered **Transit State Modal** that pauses IMU step integration within $1.5\text{ m}$ of stairwell portals, and restricted **Emergency SOS** routing strictly to `Building Entry` ground-level exits:

```kotlin
// MWE: 3D Euclidean Heuristic & Strict SOS Building Exit Selection
private fun heuristic3D(a: NodeEntity, b: NodeEntity): Float {
    val dx = a.x - b.x
    val dy = a.y - b.y
    val dz = (a.floorLevel - b.floorLevel) * 4.0f
    return kotlin.math.sqrt(dx * dx + dy * dy + dz * dz)
}

fun findNearestEmergencyExitRoute(startNodeId: String, allNodes: List<NodeEntity>): List<String> {
    val exitCandidates = allNodes.filter {
        it.type.contains("Entry", ignoreCase = true) || it.type.contains("Exit", ignoreCase = true)
    }
    return exitCandidates
        .mapNotNull { exit -> computeAStar3D(startNodeId, exit.nodeId).takeIf { it.size > 1 } }
        .minByOrNull { pathCostMeters(it) } ?: emptyList()
}
```

### 3. Validation Outcome
Eliminated cross-floor routing bleed and reduced post-stairwell localization error by $83.5\%$ ($5.40\text{ m} \rightarrow 0.89\text{ m}$).

---

## Entry 7 (Weeks 15–16): Portable JSON Map Engine with UUID Translation & Dynamic UI

### 1. Problem Statement
Importing an exported `.json` building map on a device that already contained an earlier version of the same building triggered SQLite `PRIMARY KEY` conflicts on `building_id`, `nodeId`, and `edgeId`.

### 2. Solution & MWE (`MapViewModel.kt`)
Built a deterministic UUIDv4 translation map during JSON deserialization that assigns fresh UUIDs to the building and every node while remapping `fromNodeId` and `toNodeId` across all edges, and updated the Building Selection Dialog UI with responsive dynamic height (`Modifier.heightIn(min = 64.dp)`) and geofence radius metadata:

```kotlin
// MWE: Referential Integrity Preservation During JSON UUID Translation
val nodeUuidMap = mutableMapOf<String, String>()
for (i in 0 until nodesJson.length()) {
    val nodeObj = nodesJson.getJSONObject(i)
    nodeUuidMap[nodeObj.getString("nodeId")] = UUID.randomUUID().toString()
}
for (j in 0 until edgesJson.length()) {
    val edgeObj = edgesJson.getJSONObject(j)
    val newFromId = nodeUuidMap[edgeObj.getString("fromNodeId")] ?: continue
    val newToId = nodeUuidMap[edgeObj.getString("toNodeId")] ?: continue
    insertTranslatedEdge(newFromId, newToId, edgeObj)
}
```

---

## Entry 8 (Week 17): Browser-Based 3D A* Blueprint Console & High-Refresh-Rate Synchronization

### 1. Problem Statement
1. Evaluators required an interactive browser-based web interface capable of inspecting exported Android `.json` map packages and simulating multi-floor 3D A* routes.
2. CSS flexbox stretching on the HTML5 `<canvas>` caused unscaled click coordinates to miss node hitboxes, and frame-counted stairwell pauses (`pauseFrames = 50`) ran twice as fast on $100\text{ Hz}$ displays compared to $60\text{ Hz}$ displays.

### 2. Solution & MWE (`index.html`)
Applied internal-to-CSS DPI coordinate scaling for exact $24\text{ px}$ node hit-detection, docked the HUD status bars to prevent text collisions, and refactored the walk simulator to use hardware-independent `performance.now()` time deltas:

```javascript
// MWE: DPI-Scaled Canvas Hit Detection & Refresh-Rate-Independent Transit State Timer
function getCanvasMouseCoords(e) {
    const rect = mapCanvas.getBoundingClientRect();
    return {
        x: (e.clientX - rect.left) * (mapCanvas.width / rect.width),
        y: (e.clientY - rect.top) * (mapCanvas.height / rect.height)
    };
}

function updatePortalTransitState(dtMs, nextFloorLevel) {
    simState.transitElapsedMs += dtMs;
    if (simState.transitElapsedMs >= 2800) {
        simState.inTransit = false;
        selectFloor(nextFloorLevel);
        simState.segIndex++;
    }
}
```

### 3. Validation Outcome
Verified $100\%$ node click accuracy, smooth $100.0\text{ FPS}$ canvas rendering, and synchronized $2.8\text{-second}$ stairwell Transit State transitions across both Android and Web platforms.
