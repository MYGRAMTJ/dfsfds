<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Управление машиной руками</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/vision_bundle.mjs" crossorigin="anonymous"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #0c0c0c 0%, #1a1a2e 50%, #16213e 100%);
            color: #e0e0e0;
            height: 100vh;
            overflow: hidden;
            position: relative;
        }
        .container {
            display: flex;
            height: 100vh;
        }
        #videoContainer {
            flex: 1;
            position: relative;
            background: #000;
        }
        #video {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }
        #canvas {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
        }
        .controls {
            flex: 1;
            padding: 40px;
            display: flex;
            flex-direction: column;
            justify-content: center;
            max-width: 500px;
        }
        .status {
            background: rgba(255,255,255,0.1);
            padding: 20px;
            border-radius: 12px;
            margin-bottom: 30px;
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255,255,255,0.2);
        }
        .status h2 {
            margin-bottom: 10px;
            color: #00ff88;
        }
        .gesture-info {
            font-size: 18px;
            margin-bottom: 10px;
        }
        .car-display {
            flex: 1;
            background: rgba(255,255,255,0.05);
            border-radius: 12px;
            margin-bottom: 30px;
            border: 2px solid rgba(255,255,255,0.1);
            position: relative;
            overflow: hidden;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 24px;
            backdrop-filter: blur(10px);
        }
        .car {
            font-size: 80px;
            transition: all 0.3s ease;
            position: relative;
        }
        .car.forward {
            color: #00ff88;
            transform: translateX(50px) scale(1.1);
        }
        .car.stopped {
            color: #ff4444;
            transform: scale(1);
        }
        .car-info {
            position: absolute;
            bottom: 20px;
            left: 20px;
            background: rgba(0,0,0,0.5);
            padding: 10px;
            border-radius: 8px;
            font-size: 14px;
        }
        #startBtn {
            background: linear-gradient(45deg, #00ff88, #00cc66);
            color: #000;
            border: none;
            padding: 20px 40px;
            border-radius: 50px;
            font-size: 20px;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s ease;
            align-self: center;
            max-width: 300px;
        }
        #startBtn:hover {
            transform: translateY(-3px);
            box-shadow: 0 10px 30px rgba(0,255,136,0.4);
        }
        #startBtn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
            transform: none;
        }
        .speed-bar {
            width: 100%;
            height: 20px;
            background: rgba(0,0,0,0.5);
            border-radius: 10px;
            margin-top: 10px;
            overflow: hidden;
        }
        .speed-fill {
            height: 100%;
            background: linear-gradient(90deg, #ff4444 0%, #ffaa00 50%, #00ff88 100%);
            width: 0%;
            transition: width 0.3s ease;
        }
    </style>
</head>
<body>
    <div class="container">
        <div id="videoContainer">
            <video id="video" autoplay playsinline></video>
            <canvas id="canvas"></canvas>
        </div>
        <div class="controls">
            <div class="status">
                <h2>🚗 Управление машиной</h2>
                <div id="gestureStatus" class="gesture-info">Поместите руку в кадр</div>
                <div id="handPresence" class="gesture-info">Рука не обнаружена</div>
                <div id="speedInfo">Скорость: 0 км/ч</div>
                <div class="speed-bar">
                    <div class="speed-fill" id="speedFill"></div>
                </div>
            </div>
            <div class="car-display">
                <div class="car" id="carIcon">🚗</div>
                <div class="car-info">
                    <div>Статус: <span id="carStatus">Остановлена</span></div>
                    <div id="positionInfo">Позиция: 0 м</div>
                </div>
            </div>
            <button id="startBtn">🚀 Запустить симуляцию</button>
        </div>
    </div>

    <script>
        const video = document.getElementById('video');
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        const startBtn = document.getElementById('startBtn');
        const gestureStatus = document.getElementById('gestureStatus');
        const handPresence = document.getElementById('handPresence');
        const speedInfo = document.getElementById('speedInfo');
        const speedFill = document.getElementById('speedFill');
        const carIcon = document.getElementById('carIcon');
        const carStatus = document.getElementById('carStatus');
        const positionInfo = document.getElementById('positionInfo');

        let handLandmarker = null;
        let runningMode = "VIDEO";
        let enableGesture = false;
        let isHandOpen = false; // true = вперед (ладонь), false = тормоз (кулак)
        let carSpeed = 0;
        let carPosition = 0;
        let lastPalmX = 0.5;
        const MAX_SPEED = 100;
        const ACCELERATION = 2;
        const FRICTION = 1.5;
        const SMOOTH_FACTOR = 0.6;
        const FIST_THRESHOLD = 0.03;
        const OPEN_HAND_THRESHOLD = 0.12; // Порог для открытой ладони

        // Инициализация MediaPipe
        async function initMediaPipe() {
            const vision = await FilesetResolver.forVisionTasks(
                "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm"
            );
            handLandmarker = await HandLandmarker.createFromOptions(vision, {
                baseOptions: {
                    modelAssetPath: `https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task`,
                    delegate: "GPU"
                },
                runningMode: runningMode,
                numHands: 1,
                minHandDetectionConfidence: 0.7,
                minTrackingConfidence: 0.7
            });
        }

        // Запуск камеры
        async function startCamera() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { width: 1280, height: 720 }
                });
                video.srcObject = stream;
                video.onloadedmetadata = () => {
                    canvas.width = video.videoWidth;
                    canvas.height = video.videoHeight;
                    gameLoop();
                };
            } catch (err) {
                alert('Ошибка доступа к камере: ' + err.message);
            }
        }

        // Игровой цикл
        function gameLoop() {
            async function detectHands() {
                if (!handLandmarker || !enableGesture) {
                    requestAnimationFrame(detectHands);
                    return;
                }
                const results = handLandmarker.detectForVideo(video, performance.now());
                drawResults(results);
                processHandGestures(results);
                updateCar();
                requestAnimationFrame(detectHands);
            }
            detectHands();
        }

        // Отрисовка
        function drawResults(results) {
            ctx.save();
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = isHandOpen ? '#00ff88' : '#ff4444';
            ctx.lineWidth = 4;

            if (results.landmarks) {
                for (const landmarks of results.landmarks) {
                    drawConnectors(ctx, landmarks, HAND_CONNECTIONS, {color: isHandOpen ? '#00ff88' : '#ff4444', lineWidth: 4});
                    drawLandmarks(ctx, landmarks, {color: isHandOpen ? '#00ff88' : '#ff4444', lineWidth: 3, radius: 4});
                }
            }
            ctx.restore();
        }

        // Обработка жестов
        function processHandGestures(results) {
            if (!results.landmarks || results.landmarks.length === 0) {
                handPresence.textContent = 'Рука не обнаружена';
                isHandOpen = false;
                return;
            }

            handPresence.textContent = 'Рука обнаружена ✅';
            const landmarks = results.landmarks[0];

            // Позиция центра ладони (среднее MCP точек пальцев: 5,9,13,17)
            const palmPoints = [5,9,13,17];
            let palmX = 0, palmY = 0;
            for (const pt of palmPoints) {
                palmX += landmarks[pt].x;
                palmY += landmarks[pt].y;
            }
            palmX /= palmPoints.length;
            palmY /= palmPoints.length;

            lastPalmX = lastPalmX * SMOOTH_FACTOR + palmX * (1 - SMOOTH_FACTOR);

            // Детекция жеста
            const isOpenNow = detectOpenHand(landmarks);
            isHandOpen = isOpenNow;

            gestureStatus.textContent = isHandOpen ? 
                `Ладонь открыта → Газ (Скорость: ${Math.round(carSpeed)})` : 
                `Муšt (кулак) → Тормоз`;

            ctx.fillStyle = isHandOpen ? '#00ff88' : '#ff4444';
            ctx.beginPath();
            ctx.arc(palmX * canvas.width, palmY * canvas.height, 15, 0, 2 * Math.PI);
            ctx.fill();
        }

        // Детекция открытой ладони vs кулак
        function detectOpenHand(landmarks) {
            const fingerTips = [4, 8, 12, 16, 20]; // Кончики пальцев
            const fingerPIPs = [3, 6, 10, 14, 18]; // PIP суставы
            let totalDist = 0;

            for (let i = 0; i < fingerTips.length; i++) {
                const tip = landmarks[fingerTips[i]];
                const pip = landmarks[fingerPIPs[i]];
                const dx = tip.x - pip.x;
                const dy = tip.y - pip.y;
                totalDist += Math.sqrt(dx*dx + dy*dy);
            }

            const avgDist = totalDist / fingerTips.length;
            return avgDist > OPEN_HAND_THRESHOLD; // Длинные пальцы = открытая ладонь
        }

        // Обновление машины
        function updateCar() {
            if (isHandOpen) {
                // Газ
                carSpeed = Math.min(carSpeed + ACCELERATION, MAX_SPEED);
                carPosition += carSpeed * 0.01;
            } else {
                // Тормоз
                carSpeed = Math.max(carSpeed - FRICTION, 0);
            }

            // UI обновление
            speedInfo.textContent = `Скорость: ${Math.round(carSpeed)} км/ч`;
            speedFill.style.width = (carSpeed / MAX_SPEED * 100) + '%';
            carStatus.textContent = carSpeed > 5 ? 'Движется' : 'Остановлена';
            positionInfo.textContent = `Позиция: ${Math.round(carPosition)} м`;

            carIcon.className = carSpeed > 5 ? 'car forward' : 'car stopped';
        }

        // Подключения руки
        const HAND_CONNECTIONS = [
            [0,1],[1,2],[2,3],[3,4],
            [0,5],[5,6],[6,7],[7,8],
            [5,9],[9,10],[10,11],[11,12],
            [9,13],[13,14],[14,15],[15,16],
            [13,17],[17,18],[18,19],[19,20],
            [0,17]
        ];

        // Функции отрисовки
        function drawConnectors(ctx, landmarks, connections, options) {
            ctx.lineWidth = options.lineWidth;
            ctx.strokeStyle = options.color;
            for (const conn of connections) {
                const p1 = landmarks[conn[0]];
                const p2 = landmarks[conn[1]];
                ctx.beginPath();
                ctx.moveTo(p1.x * canvas.width, p1.y * canvas.height);
                ctx.lineTo(p2.x * canvas.width, p2.y * canvas.height);
                ctx.stroke();
            }
        }

        function drawLandmarks(ctx, landmarks, options) {
            for (const lm of landmarks) {
                ctx.beginPath();
                ctx.arc(lm.x * canvas.width, lm.y * canvas.height, options.radius, 0, 2 * Math.PI);
                ctx.fillStyle = options.color;
                ctx.fill();
            }
        }

        // Старт
        startBtn.addEventListener('click', async () => {
            startBtn.disabled = true;
            startBtn.textContent = 'Инициализация...';
            await initMediaPipe();
            await startCamera();
            enableGesture = true;
            startBtn.style.display = 'none';
        });
    </script>
</body>
</html>
