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
            max-width: 600px; 
            height: 600px; 
            margin: 20px 0; 
        }
        #scanner-container { 
            position: relative; 
            width: 100%; 
            max-width: 400px; 
            height: 0; 
            padding-bottom: 75%; 
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
    <!-- ... (структура интерфейса остается без изменений) ... -->

    <script src="https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/jsqr@1.3.2/dist/jsQR.min.js"></script>
    <script>
        // ... (все переменные и функции остаются без изменений) ...

        async function generateQR() {
            const inputText = document.getElementById('qrInput').value;
            if (!inputText) return alert('Введите текст для генерации');

            const qr = qrcode(0, 'H'); // Высокий уровень коррекции ошибок
            qr.addData(inputText);
            qr.make();

            const qrContainer = document.getElementById('qr-container');
            qrContainer.innerHTML = qr.createImgTag({ 
                ecLevel: 'H', // Уровень коррекции ошибок (H — высокий)
                size: 200 // Размер изображения в пикселях (можно увеличить)
            });
        }

        async function startScanner() {
            try {
                const constraints = {
                    video: {
                        facingMode: 'environment', // Камера задней стороны
                        width: { min: 640 },
                        height: { min: 480 }
                    }
                };

                mediaStream = await navigator.mediaDevices.getUserMedia(constraints);
                video.srcObject = mediaStream;
                video.play();

                video.addEventListener('play', () => {
                    canvas.width = video.videoWidth;
                    canvas.height = video.videoHeight;
                    console.log('Canvas size:', canvas.width, 'x', canvas.height);
                    console.log('Video size:', video.videoWidth, 'x', video.videoHeight);
                });

                scannerInterval = setInterval(() => {
                    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
                    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
                    let code = jsQR(imageData.data, imageData.width, imageData.height);

                    // Попытка повернуть изображение на 90, 180, 270 градусов
                    if (!code) {
                        // Поворот на 90 градусов
                        const rotatedData90 = rotateImage(imageData.data, imageData.width, imageData.height, 90);
                        code = jsQR(rotatedData90, imageData.height, imageData.width);

                        if (!code) {
                            // Поворот на 180 градусов
                            const rotatedData180 = rotateImage(imageData.data, imageData.width, imageData.height, 180);
                            code = jsQR(rotatedData180, imageData.width, imageData.height);
                        }
                    }

                    // Вывод отладочной информации
                    console.log('Processing frame...');
                    if (code) {
                        console.log('QR detected:', code.data);
                        clearInterval(scannerInterval);
                        mediaStream.getTracks().forEach(track => track.stop());
                        document.getElementById('result').innerText = `Результат: ${code.data}`;
                    } else {
                        console.log('No QR detected');
                    }
                }, 150); // Уменьшен интервал для мобильных устройств
            } catch (error) {
                console.error('Ошибка доступа к камере:', error);
                alert('Не удалось получить доступ к камере. Убедитесь, что вы разрешили доступ к камере и открываете файл через http:// или https://');
            }
        }

        // Функция для поворота изображения
        function rotateImage(data, width, height, angle) {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            canvas.width = width;
            canvas.height = height;
            ctx.putImageData(new ImageData(new Uint8ClampedArray(data), width, height), 0, 0);

            const rotatedCanvas = document.createElement('canvas');
            const rotatedCtx = rotatedCanvas.getContext('2d');

            rotatedCanvas.width = height;
            rotatedCanvas.height = width;
            rotatedCtx.translate(rotatedCanvas.width / 2, rotatedCanvas.height / 2);
            rotatedCtx.rotate(angle * Math.PI / 180);
            rotatedCtx.drawImage(canvas, -(width / 2), -(height / 2));

            return rotatedCtx.getImageData(0, 0, rotatedCanvas.width, rotatedCanvas.height).data;
        }
    </script>
</body>
</html>
