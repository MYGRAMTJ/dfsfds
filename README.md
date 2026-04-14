<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Hand Control - Single File</title>
    
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>

    <style>
        :root {
            --bg-color: #0b0e14;
            --card-bg: #161b22;
            --accent-color: #00d2ff;
            --text-color: #e6edf3;
        }

        body {
            margin: 0;
            padding: 0;
            background-color: var(--bg-color);
            color: var(--text-color);
            font-family: 'Segoe UI', Roboto, sans-serif;
            overflow: hidden;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }

        /* Окно предпросмотра камеры */
        .output_canvas {
            position: fixed;
            bottom: 20px;
            right: 20px;
            width: 280px;
            height: 210px;
            border-radius: 15px;
            border: 2px solid var(--accent-color);
            transform: scaleX(-1); /* Зеркальное отображение */
            background: #000;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            z-index: 100;
        }

        .container {
            text-align: center;
            z-index: 10;
        }

        h1 { font-weight: 300; letter-spacing: 2px; margin-bottom: 30px; }

        .grid {
            display: flex;
            gap: 20px;
        }

        .card {
            background: var(--card-bg);
            padding: 30px;
            border-radius: 16px;
            border: 1px solid rgba(255, 255, 255, 0.1);
            width: 200px;
            transition: all 0.2s ease;
            user-select: none;
        }

        /* Состояние при наведении виртуальным курсором */
        .card.hover-effect {
            border-color: var(--accent-color);
            background: rgba(0, 210, 255, 0.05);
            transform: translateY(-5px);
        }

        .card.active {
            background: var(--accent-color);
            color: #000;
            box-shadow: 0 0 30px rgba(0, 210, 255, 0.4);
        }

        /* Стилизация виртуального курсора */
        #cursor {
            position: fixed;
            width: 20px;
            height: 20px;
            background: rgba(255, 255, 255, 0.3);
            border: 2px solid var(--accent-color);
            border-radius: 50%;
            pointer-events: none;
            z-index: 9999;
            transform: translate(-50%, -50%);
            transition: transform 0.1s ease-out, width 0.1s, height 0.1s;
        }

        #cursor.clicking {
            width: 30px;
            height: 30px;
            background: var(--accent-color);
            border-color: #fff;
        }

        .status {
            position: fixed;
            top: 20px;
            left: 20px;
            font-size: 0.9rem;
            color: #8b949e;
        }
    </style>
</head>
<body>

    <div class="status" id="status">Инициализация камеры...</div>

    <div class="container">
        <h1>AIR CONTROL SYSTEM</h1>
        <div class="grid">
            <div class="card" id="btn1">
                <h3>Сенсор 01</h3>
                <p>Сожмите пальцы</p>
            </div>
            <div class="card" id="btn2">
                <h3>Сенсор 02</h3>
                <p>Для активации</p>
            </div>
        </div>
    </div>

    <video class="input_video" style="display: none;"></video>
    <canvas class="output_canvas"></canvas>
    <div id="cursor"></div>

    <script>
        const videoElement = document.querySelector('.input_video');
        const canvasElement = document.querySelector('.output_canvas');
        const canvasCtx = canvasElement.getContext('2d');
        const cursor = document.getElementById('cursor');
        const statusText = document.getElementById('status');

        let lastClickTime = 0;

        function onResults(results) {
            // Отрисовка фида с камеры
            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.drawImage(results.image, 0, 0, canvasElement.width, canvasElement.height);

            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                statusText.innerText = "Система активна: Рука обнаружена";
                
                const landmarks = results.multiHandLandmarks[0];
                
                // Точка 8: Кончик указательного пальца (Управление курсором)
                const indexTip = landmarks[8];
                // Точка 4: Кончик большого пальца (Для определения "клика")
                const thumbTip = landmarks[4];

                // Расчет координат курсора (инверсия X для зеркального эффекта)
                const screenX = (1 - indexTip.x) * window.innerWidth;
                const screenY = indexTip.y * window.innerHeight;

                // Плавное движение курсора
                cursor.style.left = `${screenX}px`;
                cursor.style.top = `${screenY}px`;

                // Расчет расстояния между большим и указательным пальцами
                const distance = Math.hypot(indexTip.x - thumbTip.x, indexTip.y - thumbTip.y);

                // Если пальцы сведены (расстояние меньше порога)
                if (distance < 0.05) {
                    cursor.classList.add('clicking');
                    performClick(screenX, screenY);
                } else {
                    cursor.classList.remove('clicking');
                    updateHover(screenX, screenY);
                }
            } else {
                statusText.innerText = "Ожидание обнаружения руки...";
            }
            canvasCtx.restore();
        }

        // Эффект наведения
        function updateHover(x, y) {
            const cards = document.querySelectorAll('.card');
            cards.forEach(card => {
                const rect = card.getBoundingClientRect();
                if (x > rect.left && x < rect.right && y > rect.top && y < rect.bottom) {
                    card.classList.add('hover-effect');
                } else {
                    card.classList.remove('hover-effect');
                }
            });
        }

        // Эмуляция клика
        function performClick(x, y) {
            const now = Date.now();
            if (now - lastClickTime < 800) return; // Защита от частых кликов

            const elements = document.elementsFromPoint(x, y);
            elements.forEach(el => {
                if (el.classList.contains('card')) {
                    el.classList.toggle('active');
                    lastClickTime = now;
                }
            });
        }

        // Конфигурация MediaPipe Hands
        const hands = new Hands({
            locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
        });

        hands.setOptions({
            maxNumHands: 1,
            modelComplexity: 1,
            minDetectionConfidence: 0.8,
            minTrackingConfidence: 0.8
        });

        hands.onResults(onResults);

        // Запуск камеры
        const camera = new Camera(videoElement, {
            onFrame: async () => {
                await hands.send({ image: videoElement });
            },
            width: 640,
            height: 480
        });
        
        camera.start().catch(() => {
            statusText.innerText = "Ошибка: Доступ к камере запрещен";
        });
    </script>
</body>
</html>
