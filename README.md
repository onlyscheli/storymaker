<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>ANACONDA</title>
    <link href="https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap" rel="stylesheet">
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
            touch-action: manipulation;
        }
        body {
            background-color: #050b14;
            color: #00ff88;
            font-family: 'Press Start 2P', monospace;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            overflow: hidden;
            padding: 5px 0;
        }
        h1 {
            font-size: 1.1rem;
            color: #00ff88;
            text-shadow: 2px 2px #ff0055;
            margin: 0 0 6px 0;
            letter-spacing: 2px;
        }
        .ui-panel {
            display: flex;
            justify-content: space-between;
            width: 320px;
            font-size: 0.55rem;
            margin-bottom: 6px;
        }
        .ui-box {
            background: #0d1b2a;
            border: 2px solid #00ff88;
            padding: 5px 6px;
            box-shadow: inset 0 0 5px #00ff88;
        }
        #game-container {
            position: relative;
            border: 3px solid #ff0055;
            box-shadow: 0 0 15px #ff0055, inset 0 0 15px #00ff88;
            background-color: #080e18;
            width: 320px;
            height: 320px;
        }
        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }
        #message-overlay {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            width: 90%;
            pointer-events: none;
            z-index: 5;
        }
        .retro-msg {
            font-size: 0.75rem;
            color: #ffee00;
            text-shadow: 2px 2px #ff0000;
            animation: popUp 1s ease-out forwards;
            opacity: 0;
        }
        @keyframes popUp {
            0% { transform: scale(0.4); opacity: 0; }
            30% { transform: scale(1.1); opacity: 1; }
            70% { transform: scale(1); opacity: 1; }
            100% { transform: scale(1); opacity: 0; }
        }
        #start-screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(5, 11, 20, 0.95);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            text-align: center;
            padding: 20px;
            z-index: 10;
        }
        .btn {
            background-color: #ff0055;
            color: #ffffff;
            font-family: 'Press Start 2P', monospace;
            font-size: 0.85rem;
            padding: 14px 28px;
            border: 3px solid #00ff88;
            cursor: pointer;
            box-shadow: 4px 4px #00ff88;
        }
        .btn:active {
            transform: translate(2px, 2px);
            box-shadow: 2px 2px #00ff88;
        }
        /* Steuerkreuz */
        .controls {
            display: grid;
            grid-template-columns: 65px 65px 65px;
            grid-template-rows: 46px 46px;
            gap: 6px;
            margin-top: 8px;
            justify-content: center;
        }
        .ctrl-btn {
            background-color: #0d1b2a;
            border: 2px solid #00ff88;
            color: #00ff88;
            font-size: 1.1rem;
            display: flex;
            align-items: center;
            justify-content: center;
            border-radius: 6px;
            box-shadow: 0 3px 0 #005533;
        }
        .ctrl-btn:active {
            background-color: #00ff88;
            color: #050b14;
            transform: translateY(2px);
            box-shadow: 0 1px 0 #005533;
        }
        .btn-up { grid-column: 2; grid-row: 1; }
        .btn-left { grid-column: 1; grid-row: 2; }
        .btn-down { grid-column: 2; grid-row: 2; }
        .btn-right { grid-column: 3; grid-row: 2; }
    </style>
</head>
<body>

    <h1>ANACONDA</h1>
    
    <div class="ui-panel">
        <div class="ui-box">PTS: <span id="score">0</span></div>
        <div class="ui-box">LVL: <span id="level">1</span></div>
        <div class="ui-box">HI: <span id="highscore">0</span></div>
    </div>

    <div id="game-container">
        <canvas id="gameCanvas" width="400" height="400"></canvas>
        <div id="message-overlay"></div>
        
        <div id="start-screen">
            <button class="btn" id="start-btn">▶ PLAY</button>
        </div>
    </div>

    <!-- Touch-Steuerkreuz -->
    <div class="controls">
        <div class="ctrl-btn btn-up" id="btn-up">▲</div>
        <div class="ctrl-btn btn-left" id="btn-left">◀</div>
        <div class="ctrl-btn btn-down" id="btn-down">▼</div>
        <div class="ctrl-btn btn-right" id="btn-right">▶</div>
    </div>

<script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    const scoreEl = document.getElementById("score");
    const levelEl = document.getElementById("level");
    const highscoreEl = document.getElementById("highscore");
    const startScreen = document.getElementById("start-screen");
    const startBtn = document.getElementById("start-btn");
    const messageOverlay = document.getElementById("message-overlay");

    const GRID_SIZE = 20;
    const TILE_COUNT = canvas.width / GRID_SIZE;

    let snake = [];
    let food = {};
    let obstacles = [];
    let dx = 1;
    let dy = 0;
    let score = 0;
    let level = 1;
    let highscore = localStorage.getItem("anaconda_high") || 0;
    let gameInterval = null;
    let gameRunning = false;
    let speed = 160;

    highscoreEl.textContent = highscore;

    const compliments = ["⚡ FAST!", "🔥 FIRE!", "👑 KING!", "🚀 EPIC!", "💎 BONUS!", "⭐ GO!"];

    let audioCtx = null;
    function initAudio() {
        if (!audioCtx) {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        }
    }

    function playSound(type) {
        if (!audioCtx) return;
        try {
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            const now = audioCtx.currentTime;

            if (type === 'eat') {
                osc.type = 'square';
                osc.frequency.setValueAtTime(350, now);
                osc.frequency.exponentialRampToValueAtTime(700, now + 0.08);
                gain.gain.setValueAtTime(0.1, now);
                gain.gain.exponentialRampToValueAtTime(0.01, now + 0.08);
                osc.start(now);
                osc.stop(now + 0.08);
            } else if (type === 'level') {
                osc.type = 'triangle';
                osc.frequency.setValueAtTime(400, now);
                osc.frequency.setValueAtTime(650, now + 0.1);
                gain.gain.setValueAtTime(0.12, now);
                gain.gain.exponentialRampToValueAtTime(0.01, now + 0.25);
                osc.start(now);
                osc.stop(now + 0.25);
            } else if (type === 'gameover') {
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(250, now);
                osc.frequency.exponentialRampToValueAtTime(60, now + 0.35);
                gain.gain.setValueAtTime(0.15, now);
                gain.gain.exponentialRampToValueAtTime(0.01, now + 0.35);
                osc.start(now);
                osc.stop(now + 0.35);
            }
        } catch(e) {}
    }

    function showPopup(text) {
        const msg = document.createElement("div");
        msg.className = "retro-msg";
        msg.textContent = text;
        messageOverlay.appendChild(msg);
        setTimeout(() => msg.remove(), 1000);
    }

    function startGame(e) {
        if (e) e.preventDefault();
        initAudio();
        snake = [{ x: 10, y: 10 }, { x: 9, y: 10 }, { x: 8, y: 10 }];
        dx = 1;
        dy = 0;
        score = 0;
        level = 1;
        speed = 160;
        obstacles = [];
        scoreEl.textContent = score;
        levelEl.textContent = level;
        startScreen.style.display = "none";
        gameRunning = true;
        spawnFood();
        if (gameInterval) clearInterval(gameInterval);
        gameInterval = setInterval(gameLoop, speed);
    }

    function spawnFood() {
        let valid = false;
        while (!valid) {
            food = {
                x: Math.floor(Math.random() * TILE_COUNT),
                y: Math.floor(Math.random() * TILE_COUNT)
            };
            valid = true;
            for (let part of snake) {
                if (part.x === food.x && part.y === food.y) valid = false;
            }
            for (let obs of obstacles) {
                if (obs.x === food.x && obs.y === food.y) valid = false;
            }
        }
    }

    function spawnObstacles() {
        obstacles = [];
        if (level === 1) return;

        let count = (level - 1) * 2;
        for (let i = 0; i < count; i++) {
            let valid = false;
            let obs;
            while (!valid) {
                obs = {
                    x: Math.floor(Math.random() * TILE_COUNT),
                    y: Math.floor(Math.random() * TILE_COUNT)
                };
                valid = true;
                if (Math.abs(obs.x - snake[0].x) < 3 && Math.abs(obs.y - snake[0].y) < 3) valid = false;
                if (obs.x === food.x && obs.y === food.y) valid = false;
                for (let part of snake) {
                    if (part.x === obs.x && part.y === obs.y) valid = false;
                }
            }
            obstacles.push(obs);
        }
    }

    function gameLoop() {
        if (!gameRunning) return;
        update();
        draw();
    }

    function update() {
        const head = { x: snake[0].x + dx, y: snake[0].y + dy };

        if (head.x < 0 || head.x >= TILE_COUNT || head.y < 0 || head.y >= TILE_COUNT) {
            gameOver();
            return;
        }

        for (let part of snake) {
            if (head.x === part.x && head.y === part.y) {
                gameOver();
                return;
            }
        }

        for (let obs of obstacles) {
            if (head.x === obs.x && head.y === obs.y) {
                gameOver();
                return;
            }
        }

        snake.unshift(head);

        if (head.x === food.x && head.y === food.y) {
            score += 10 * level;
            scoreEl.textContent = score;
            playSound('eat');

            if (score >= level * 50) {
                level++;
                levelEl.textContent = level;
                speed = Math.max(50, speed - 15);
                clearInterval(gameInterval);
                gameInterval = setInterval(gameLoop, speed);
                spawnObstacles();
                playSound('level');
                showPopup(`LVL ${level} 🚀`);
            } else if (score % 30 === 0) {
                showPopup(compliments[Math.floor(Math.random() * compliments.length)]);
            }

            spawnFood();
        } else {
            snake.pop();
        }
    }

    function draw() {
        ctx.fillStyle = "#050b14";
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // Hindernisse
        ctx.fillStyle = "#ff0055";
        ctx.shadowBlur = 6;
        ctx.shadowColor = "#ff0055";
        for (let obs of obstacles) {
            ctx.fillRect(obs.x * GRID_SIZE + 1, obs.y * GRID_SIZE + 1, GRID_SIZE - 2, GRID_SIZE - 2);
        }

        // Futter
        ctx.fillStyle = "#ffee00";
        ctx.shadowBlur = 10;
        ctx.shadowColor = "#ffee00";
        ctx.fillRect(food.x * GRID_SIZE + 1, food.y * GRID_SIZE + 1, GRID_SIZE - 2, GRID_SIZE - 2);

        // Anaconda
        for (let i = 0; i < snake.length; i++) {
            let part = snake[i];
            ctx.fillStyle = i === 0 ? "#00ff88" : "#00aa55";
            ctx.shadowBlur = i === 0 ? 10 : 2;
            ctx.shadowColor = "#00ff88";
            ctx.fillRect(part.x * GRID_SIZE + 1, part.y * GRID_SIZE + 1, GRID_SIZE - 2, GRID_SIZE - 2);
        }
        ctx.shadowBlur = 0;
    }

    function gameOver() {
        gameRunning = false;
        clearInterval(gameInterval);
        playSound('gameover');

        if (score > highscore) {
            highscore = score;
            localStorage.setItem("anaconda_high", highscore);
            highscoreEl.textContent = highscore;
            showPopup("NEW HIGH! 🏆");
        }

        startScreen.style.display = "flex";
        startBtn.textContent = "AGAIN";
    }

    // Touch-Steuerung über Buttons
    document.getElementById("btn-up").addEventListener("click", () => { if (dy === 0) { dx = 0; dy = -1; } });
    document.getElementById("btn-down").addEventListener("click", () => { if (dy === 0) { dx = 0; dy = 1; } });
    document.getElementById("btn-left").addEventListener("click", () => { if (dx === 0) { dx = -1; dy = 0; } });
    document.getElementById("btn-right").addEventListener("click", () => { if (dx === 0) { dx = 1; dy = 0; } });

    // Zuverlässiger Start-Button (unterstützt Klick und Touch direkt)
    startBtn.addEventListener("click", startGame);
    startBtn.addEventListener("touchstart", startGame);

    // Tastatur-Steuerung (Pfeiltasten & WASD)
    window.addEventListener("keydown", e => {
        if (!gameRunning && e.code === "Space") {
            startGame();
            return;
        }
        switch (e.code) {
            case "ArrowUp":
            case "KeyW":
                if (dy === 0) { dx = 0; dy = -1; }
                break;
            case "ArrowDown":
            case "KeyS":
                if (dy === 0) { dx = 0; dy = 1; }
                break;
            case "ArrowLeft":
            case "KeyA":
                if (dx === 0) { dx = -1; dy = 0; }
                break;
            case "ArrowRight":
            case "KeyD":
                if (dx === 0) { dx = 1; dy = 0; }
                break;
        }
    });

    // Swipe-Steuerung direkt auf dem Spielfeld
    let touchStartX = 0;
    let touchStartY = 0;

    canvas.addEventListener("touchstart", e => {
        touchStartX = e.touches[0].clientX;
        touchStartY = e.touches[0].clientY;
        e.preventDefault();
    }, { passive: false });

    canvas.addEventListener("touchmove", e => { e.preventDefault(); }, { passive: false });

    canvas.addEventListener("touchend", e => {
        if (!gameRunning) return;
        let diffX = e.changedTouches[0].clientX - touchStartX;
        let diffY = e.changedTouches[0].clientY - touchStartY;

        if (Math.abs(diffX) > Math.abs(diffY)) {
            if (diffX > 25 && dx === 0) { dx = 1; dy = 0; }
            else if (diffX < -25 && dx === 0) { dx = -1; dy = 0; }
        } else {
            if (diffY > 25 && dy === 0) { dx = 0; dy = 1; }
            else if (diffY < -25 && dy === 0) { dx = 0; dy = -1; }
        }
        e.preventDefault();
    }, { passive: false });

    draw();
</script>

</body>
</html>

