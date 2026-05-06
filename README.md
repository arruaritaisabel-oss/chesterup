<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Chester Up | Real-time Monitor</title>
    <style>
        :root {
            --honey-gold: #d4a373;
            --soft-blue: #a8dadc;
            --deep-black: #0a0a0b;
            --glass-bg: rgba(255, 255, 255, 0.05);
            --alert-red: #ff4d4d;
        }

        body {
            background-color: var(--deep-black);
            color: #e0e0e0;
            font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            overflow: hidden;
        }

        /* Estética de Cristal */
        .glass-panel {
            background: var(--glass-bg);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 30px;
            padding: 30px;
            text-align: center;
            width: 85%;
            max-width: 400px;
            box-shadow: 0 20px 50px rgba(0,0,0,0.5);
        }

        h1 {
            font-weight: 300;
            letter-spacing: 5px;
            text-transform: uppercase;
            font-size: 1.2rem;
            color: var(--honey-gold);
            margin: 0 0 10px 0;
        }

        .time-display {
            font-size: 2.5rem;
            font-weight: 200;
            margin-bottom: 5px;
            color: white;
        }

        .schedule-tag {
            font-size: 0.7rem;
            color: var(--soft-blue);
            text-transform: uppercase;
            letter-spacing: 2px;
            margin-bottom: 30px;
        }

        /* Monitor Realista */
        .monitor-container {
            position: relative;
            width: 220px;
            height: 220px;
            margin: 20px auto;
        }

        .radar-circle {
            width: 100%;
            height: 100%;
            border-radius: 50%;
            border: 2px solid var(--glass-bg);
            display: flex;
            align-items: center;
            justify-content: center;
            position: relative;
            z-index: 2;
            transition: all 0.5s ease;
        }

        /* Representación visual de Chester */
        .chester-avatar {
            width: 160px;
            height: 160px;
            border-radius: 50%;
            background: linear-gradient(135deg, #d4a373 0%, #b88b5c 100%);
            border: 5px solid var(--deep-black);
            box-shadow: 0 0 30px rgba(212, 163, 115, 0.3);
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 4rem;
        }

        /* Animaciones de estado */
        .active-pulse {
            position: absolute;
            width: 100%;
            height: 100%;
            border-radius: 50%;
            border: 2px solid var(--soft-blue);
            animation: radar 2s infinite linear;
            opacity: 0;
        }

        @keyframes radar {
            0% { transform: scale(1); opacity: 0.8; }
            100% { transform: scale(1.5); opacity: 0; }
        }

        .alerting .chester-avatar {
            animation: shake 0.2s infinite;
            background: var(--alert-red);
            box-shadow: 0 0 50px var(--alert-red);
        }

        @keyframes shake {
            0% { transform: translate(1px, 1px) rotate(0deg); }
            50% { transform: translate(-1px, -2px) rotate(-1deg); }
            100% { transform: translate(1px, 1px) rotate(0deg); }
        }

        /* Botones */
        .btn {
            margin-top: 30px;
            padding: 18px 40px;
            border-radius: 50px;
            border: none;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s;
            width: 100%;
            text-transform: uppercase;
            letter-spacing: 1px;
        }

        .btn-primary {
            background: var(--honey-gold);
            color: var(--deep-black);
        }

        .btn-stop {
            background: #ff4d4d;
            color: white;
            display: none;
        }

        #video-feed {
            position: absolute;
            bottom: -100px;
            opacity: 0;
        }

        canvas { display: none; }
        
        .status-label {
            margin-top: 15px;
            font-size: 0.8rem;
            color: var(--soft-blue);
            font-weight: bold;
        }
    </style>
</head>
<body>

    <div class="glass-panel">
        <h1>Chester Up</h1>
        <div id="clock" class="time-display">00:00:00</div>
        <div class="schedule-tag">Activación: 05:00 AM (ART)</div>

        <div class="monitor-container" id="monitor-ui">
            <div id="pulse-ring" class=""></div>
            <div class="radar-circle">
                <div class="chester-avatar">🐶</div>
            </div>
        </div>

        <div id="status" class="status-label">SISTEMA APAGADO</div>

        <button id="startBtn" class="btn btn-primary">Iniciar Vigilancia</button>
        <button id="stopBtn" class="btn btn-stop">¡Estoy despierto!</button>
    </div>

    <video id="video-feed" autoplay playsinline muted></video>
    <canvas id="proc-canvas"></canvas>

<script>
    const clock = document.getElementById('clock');
    const startBtn = document.getElementById('startBtn');
    const stopBtn = document.getElementById('stopBtn');
    const statusLabel = document.getElementById('status');
    const monitorUI = document.getElementById('monitor-ui');
    const pulseRing = document.getElementById('pulse-ring');
    const video = document.getElementById('video-feed');
    const canvas = document.getElementById('proc-canvas');
    const ctx = canvas.getContext('2d', { willReadFrequently: true });

    let isArmed = false;
    let isVigilating = false;
    let isAlerting = false;
    let motionTimer = 0;
    let lastData = null;
    let alertInterval = null;

    // Reloj y Lógica 5 AM Buenos Aires
    setInterval(() => {
        const now = new Date();
        clock.innerText = now.toLocaleTimeString('es-AR', { hour12: false });

        if (isArmed && !isVigilating) {
            if (now.getHours() === 5 && now.getMinutes() >= 0) {
                startScanning();
            }
        }
    }, 1000);

    async function initSystem() {
        try {
            const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
            video.srcObject = stream;
            isArmed = true;
            startBtn.style.display = 'none';
            statusLabel.innerText = "ARMADO: SE ACTIVARÁ A LAS 5:00 AM";
            statusLabel.style.color = "var(--honey-gold)";
            
            // Permitir audio (Safari/Chrome Mobile unlock)
            window.speechSynthesis.speak(new SpeechSynthesisUtterance(""));
        } catch (e) {
            alert("Error: Se requiere acceso a la cámara.");
        }
    }

    function startScanning() {
        isVigilating = true;
        pulseRing.classList.add('active-pulse');
        statusLabel.innerText = "VIGILANCIA ACTIVA";
        statusLabel.style.color = "var(--soft-blue)";
        processFrame();
    }

    function processFrame() {
        if (!isVigilating) return;

        canvas.width = 120;
        canvas.height = 90;
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
        const frame = ctx.getImageData(0, 0, canvas.width, canvas.height).data;

        if (lastData) {
            let diff = 0;
            for (let i = 0; i < frame.length; i += 4) {
                // Analizamos el canal rojo y verde para mayor precisión en penumbra
                if (Math.abs(frame[i] - lastData[i]) > 35) diff++;
            }

            if (diff > (canvas.width * canvas.height * 0.05)) {
                motionTimer++;
            } else {
                motionTimer = Math.max(0, motionTimer - 1);
            }

            // Alerta si hay movimiento durante 5 segundos (aprox 50 ciclos de 100ms)
            if (motionTimer > 50 && !isAlerting) {
                triggerAlert();
            }
        }
        lastData = frame;
        setTimeout(() => requestAnimationFrame(processFrame), 100);
    }

    function triggerAlert() {
        isAlerting = true;
        monitorUI.classList.add('alerting');
        stopBtn.style.display = 'block';
        statusLabel.innerText = "¡MOVIMIENTO DETECTADO!";
        statusLabel.style.color = "var(--alert-red)";

        const speak = () => {
            if (!isAlerting) return;
            const msg = new SpeechSynthesisUtterance("¡Atención! Chester quiere ir al baño.");
            msg.lang = 'es-AR';
            msg.rate = 0.85;
            window.speechSynthesis.speak(msg);
        };

        speak();
        alertInterval = setInterval(speak, 15000);
    }

    startBtn.addEventListener('click', initSystem);

    stopBtn.addEventListener('click', () => {
        isAlerting = false;
        clearInterval(alertInterval);
        window.speechSynthesis.cancel();
        monitorUI.classList.remove('alerting');
        stopBtn.style.display = 'none';
        statusLabel.innerText = "VIGILANCIA ACTIVA";
        statusLabel.style.color = "var(--soft-blue)";
        motionTimer = 0;
    });

</script>
</body>
</html>
