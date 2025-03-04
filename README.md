<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>QR Generator & Scanner</title>
    <style>
        body { 
            font-family: Arial, sans-serif; 
            padding: 20px; 
            margin: 0; 
            box-sizing: border-box;
        }
        .tabs { 
            margin-bottom: 20px; 
            text-align: center;
        }
        .tab { 
            display: inline-block; 
            margin: 0 10px; 
            cursor: pointer; 
            padding: 8px 15px; 
            border: 1px solid #ccc; 
            border-radius: 5px;
        }
        .tab.active { 
            background: #007bff; 
            color: white; 
        }
        .tab-content { 
            display: none; 
        }
        .tab-content.active { 
            display: block; 
        }
        #qr-container { 
            width: 100%; 
            max-width: 300px; 
            height: 300px; 
            margin: 20px 0; 
        }
        #scanner-container { 
            position: relative; 
            width: 100%; 
            max-width: 400px; 
            height: 0; 
            padding-bottom: 75%; /* Соотношение сторон 4:3 */
            margin: 20px 0;
        }
        #scanner-video { 
            width: 100%; 
            height: 100%; 
            position: absolute; 
            top: 0; 
            left: 0; 
        }
        #scanner-canvas { 
            display: none; 
        }
        #result { 
            margin-top: 10px; 
            font-weight: bold; 
            text-align: center;
        }
        /* Адаптивные стили для мобильных */
        @media (max-width: 768px) {
            #qr-container, #scanner-container {
                max-width: 100%;
            }
            .tab {
                padding: 6px 10px;
                font-size: 0.9em;
            }
        }
    </style>
</head>
<body>
    <div class="tabs">
        <div class="tab active" onclick="showTab('generate')">Создать QR</div>
        <div class="tab" onclick="showTab('scan')">Сканировать QR</div>
    </div>

    <div id="generate" class="tab-content active">
        <input type="text" id="qrInput" placeholder="Введите текст для QR" style="width: 100%; padding: 8px; margin-bottom: 10px;">
        <button onclick="generateQR()">Создать QR</button>
        <div id="qr-container"></div>
    </div>

    <div id="scan" class="tab-content">
        <button onclick="startScanner()">Включить камеру</button>
        <div id="scanner-container">
            <video id="scanner-video" autoplay></video>
            <canvas id="scanner-canvas" style="display: none;"></canvas>
        </div>
        <div id="result"></div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/jsqr@1.3.2/dist/jsQR.min.js"></script>
    <script>
        let mediaStream;
        let scannerInterval;
        const video = document.getElementById('scanner-video');
        const canvas = document.getElementById('scanner-canvas');
        const ctx = canvas.getContext('2d');

        // Функция для адаптивного изменения размеров canvas
        function resizeCanvas() {
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            // Убедимся, что canvas масштабируется правильно
            const scannerContainer = document.getElementById('scanner-container');
            scannerContainer.style.paddingBottom = (video.videoHeight / video.videoWidth) * 100 + '%';
        }

        function showTab(tabName) {
            document.querySelectorAll('.tab').forEach(tab => tab.classList.remove('active'));
            document.querySelectorAll('.tab-content').forEach(content => content.classList.remove('active'));
            
            document.querySelector(`.tab[onclick*="${tabName}"]`).classList.add('active');
            document.getElementById(tabName).classList.add('active');
        }

        async function generateQR() {
            const inputText = document.getElementById('qrInput').value;
            if (!inputText) return alert('Введите текст для генерации');

            const qr = qrcode(0, 'M');
            qr.addData(inputText);
            qr.make();

            const qrContainer = document.getElementById('qr-container');
            qrContainer.innerHTML = qr.createImgTag();
        }

        async function startScanner() {
            try {
                // Запрос доступа к камере (задняя камера)
                const constraints = {
                    video: {
                        facingMode: 'environment', // Камера задней стороны
                        width: { min: 640 }, // Минимальная ширина
                        height: { min: 480 } // Минимальная высота
                    }
                };

                mediaStream = await navigator.mediaDevices.getUserMedia(constraints);
                video.srcObject = mediaStream;
                video.play();

                // Обновляем размеры canvas при изменении размеров видео
                video.addEventListener('play', () => {
                    resizeCanvas();
                    // Учет поворота камеры на мобильных устройствах
                    const orientation = video.videoWidth > video.videoHeight ? 90 : 0;
                    if (orientation > 0) {
                        canvas.width = video.videoHeight;
                        canvas.height = video.videoWidth;
                    }
                });

                // Обновляем размеры canvas при изменении ориентации устройства
                window.addEventListener('resize', resizeCanvas);

                scannerInterval = setInterval(() => {
                    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
                    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
                    let code = jsQR(imageData.data, imageData.width, imageData.height);

                    // Если QR не найден, попробуем повернуть изображение на 180 градусов (для мобильных)
                    if (!code) {
                        // Поворот изображения на 180 градусов
                        const rotatedData = new Uint8ClampedArray(imageData.data.length);
                        const width = imageData.width;
                        const height = imageData.height;
                        for (let y = 0; y < height; y++) {
                            for (let x = 0; x < width; x++) {
                                const srcIndex = (y * width + x) * 4;
                                const dstIndex = ((height - 1 - y) * width + (width - 1 - x)) * 4;
                                rotatedData[dstIndex] = imageData.data[srcIndex];
                                rotatedData[dstIndex + 1] = imageData.data[srcIndex + 1];
                                rotatedData[dstIndex + 2] = imageData.data[srcIndex + 2];
                                rotatedData[dstIndex + 3] = imageData.data[srcIndex + 3];
                            }
                        }
                        code = jsQR(rotatedData, width, height);
                    }

                    if (code) {
                        clearInterval(scannerInterval);
                        mediaStream.getTracks().forEach(track => track.stop());
                        document.getElementById('result').innerText = `Результат: ${code.data}`;
                    }
                }, 150); // Уменьшен интервал для мобильных устройств
            } catch (error) {
                console.error('Ошибка доступа к камере:', error);
                alert('Не удалось получить доступ к камере. Убедитесь, что вы разрешили доступ к камере и открываете файл через http:// или https://');
            }
        }

        // Уничтожаем поток при выходе из приложения
        window.addEventListener('beforeunload', () => {
            if (mediaStream) {
                mediaStream.getTracks().forEach(track => track.stop());
            }
        });
    </script>
</body>
</html>
