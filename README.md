<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Driver Drowsiness Alert System</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js" crossorigin="anonymous"></script>
    <!-- Optional: MQTT.js for ESP32 hardware integration -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mqtt/4.3.7/mqtt.min.js"></script>
    <style>
        body { font-family: system-ui, sans-serif; background: #111; color: #fff; display: flex; flex-direction: column; align-items: center; margin: 0; padding: 20px; transition: background 0.3s; }
        body.alert { background: #500; } /* Flashes red when drowsy */
        .container { position: relative; width: 640px; height: 480px; margin-top: 20px; }
        video, canvas { border-radius: 8px; border: 2px solid #333; position: absolute; top: 0; left: 0; transform: scaleX(-1); }
        .dashboard { display: grid; grid-template-columns: repeat(2, 1fr); gap: 20px; width: 640px; margin-top: 20px; }
        .panel { background: #1e1e1e; padding: 20px; border-radius: 8px; border: 1px solid #333; text-align: center; }
        .status { font-size: 2rem; font-weight: bold; color: #00ffcc; }
        .status.danger { color: #ff3333; animation: blink 0.5s infinite; }
        .metric { font-size: 1.5rem; font-family: monospace; color: #aaa; margin-top: 10px; }
        button { padding: 10px 20px; font-size: 1.2rem; cursor: pointer; background: #007bff; color: white; border: none; border-radius: 5px; margin-bottom: 10px; }
        
        @keyframes blink { 50% { opacity: 0.5; } }
    </style>
</head>
<body>

    <h2>Drowsiness Detection Dashboard</h2>
    <!-- Browsers require user interaction before playing audio -->
    <button id="startBtn">Start System (Enable Audio & Camera)</button>

    <div class="container">
        <video id="input_video" width="640" height="480" autoplay playsinline></video>
        <canvas id="output_canvas" width="640" height="480"></canvas>
    </div>

    <div class="dashboard">
        <div class="panel">
            <h3>Driver Status</h3>
            <div id="driverStatus" class="status">AWAKE</div>
        </div>
        <div class="panel">
            <h3>Eye Aspect Ratio (EAR)</h3>
            <div id="earValue" class="metric">0.00</div>
        </div>
    </div>

    <script>
        const videoElement = document.getElementById('input_video');
        const canvasElement = document.getElementById('output_canvas');
        const canvasCtx = canvasElement.getContext('2d');
        const statusElement = document.getElementById('driverStatus');
        const earElement = document.getElementById('earValue');
        
        // --- Tuning Parameters ---
        const EAR_THRESHOLD = 0.22; // Adjust based on camera angle
        const DROWSY_FRAMES_LIMIT = 20; // At 30fps, ~0.6 seconds of closed eyes
        let closedFramesCount = 0;
        let isAlarmActive = false;

        // --- Audio Setup (Browser Buzzer) ---
        const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        let oscillator = null;

        function playBuzzer() {
            if (isAlarmActive) return;
            isAlarmActive = true;
            document.body.classList.add('alert');
            statusElement.innerText = "DROWSY ALARM!";
            statusElement.classList.add('danger');
            
            // Generate an annoying beep
            oscillator = audioCtx.createOscillator();
            const gainNode = audioCtx.createGain();
            oscillator.type = 'square';
            oscillator.frequency.setValueAtTime(800, audioCtx.currentTime); // 800Hz beep
            oscillator.connect(gainNode);
            gainNode.connect(audioCtx.destination);
            oscillator.start();

            // Hardware Trigger (e.g., via MQTT to ESP32)
            // if(mqttClient) mqttClient.publish('vehicle/driver/alert', 'ON');
        }

        function stopBuzzer() {
            if (!isAlarmActive) return;
            isAlarmActive = false;
            document.body.classList.remove('alert');
            statusElement.innerText = "AWAKE";
            statusElement.classList.remove('danger');
            
            if (oscillator) {
                oscillator.stop();
                oscillator.disconnect();
            }
            
            // Hardware Trigger OFF
            // if(mqttClient) mqttClient.publish('vehicle/driver/alert', 'OFF');
        }

        // --- Calculate 3D distance between two points ---
        function calculateDistance(p1, p2) {
            return Math.sqrt(Math.pow(p1.x - p2.x, 2) + Math.pow(p1.y - p2.y, 2) + Math.pow(p1.z - p2.z, 2));
        }

        function calculateEAR(landmarks, indices) {
            // indices: [p1, p2, p3, p4, p5, p6]
            // p1, p4 are horizontal. p2, p6 and p3, p5 are vertical.
            const p1 = landmarks[indices[0]], p2 = landmarks[indices[1]], p3 = landmarks[indices[2]];
            const p4 = landmarks[indices[3]], p5 = landmarks[indices[4]], p6 = landmarks[indices[5]];
            
            const v1 = calculateDistance(p2, p6);
            const v2 = calculateDistance(p3, p5);
            const h = calculateDistance(p1, p4);
            
            return (v1 + v2) / (2.0 * h);
        }

        const faceMesh = new FaceMesh({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`});
        faceMesh.setOptions({ maxNumFaces: 1, refineLandmarks: true, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5 });
        faceMesh.onResults(onResults);

        function onResults(results) {
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.drawImage(results.image, 0, 0, canvasElement.width, canvasElement.height);

            if (results.multiFaceLandmarks && results.multiFaceLandmarks.length > 0) {
                const landmarks = results.multiFaceLandmarks[0];

                // Standard MediaPipe Eye Landmark Indices
                const leftEyeIndices = [33, 160, 158, 133, 153, 144];
                const rightEyeIndices = [362, 385, 387, 263, 373, 380];

                const leftEAR = calculateEAR(landmarks, leftEyeIndices);
                const rightEAR = calculateEAR(landmarks, rightEyeIndices);
                const avgEAR = (leftEAR + rightEAR) / 2.0;

                earElement.innerText = avgEAR.toFixed(3);

                // Drowsiness State Machine
                if (avgEAR < EAR_THRESHOLD) {
                    closedFramesCount++;
                    if (closedFramesCount >= DROWSY_FRAMES_LIMIT) {
                        playBuzzer();
                    }
                } else {
                    closedFramesCount = 0;
                    stopBuzzer();
                }

                // Draw Eye Contours
                canvasCtx.fillStyle = (avgEAR < EAR_THRESHOLD) ? "red" : "lime";
                [...leftEyeIndices, ...rightEyeIndices].forEach(index => {
                    const pt = landmarks[index];
                    canvasCtx.beginPath();
                    canvasCtx.arc(pt.x * canvasElement.width, pt.y * canvasElement.height, 2, 0, 2 * Math.PI);
                    canvasCtx.fill();
                });
            }
        }

        document.getElementById('startBtn').addEventListener('click', () => {
            audioCtx.resume(); // Required to unlock Web Audio API
            const camera = new Camera(videoElement, {
                onFrame: async () => { await faceMesh.send({image: videoElement}); },
                width: 640, height: 480
            });
            camera.start();
            document.getElementById('startBtn').style.display = 'none';
        });
    </script>
</body>
</html>
