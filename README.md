<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Pro Penalty: Ultimate Striker Enhanced</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Teko:wght@400;600&display=swap');

        body {
            margin: 0;
            padding: 0;
            background-color: #1a1a1a;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            font-family: 'Teko', sans-serif;
            overflow: hidden;
            touch-action: none;
        }

        #game-container {
            position: relative;
            width: 1000px;
            max-width: 100%;
            height: 700px;
            max-height: 100vh;
            box-shadow: 0 0 50px rgba(0,0,0,0.8);
            border-radius: 4px;
            overflow: hidden;
            background: radial-gradient(circle at center top, #2c3e50, #000000);
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        #ui-layer {
            position: absolute;
            top: 20px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            color: white;
            text-transform: uppercase;
            text-shadow: 2px 2px 0px #000;
            pointer-events: none;
            z-index: 20;
        }

        .stat-box {
            background: rgba(0, 0, 0, 0.6);
            border-left: 5px solid #ffcc00;
            padding: 5px 20px;
            font-size: 24px;
            backdrop-filter: blur(5px);
            margin-bottom: 5px;
        }

        .score-big {
            font-size: 30px;
            color: #ffcc00;
        }

        .streak-container {
            display: flex;
            flex-direction: column;
            align-items: flex-end;
        }

        .streak-box {
            font-size: 20px;
            color: #00ff00;
            text-shadow: 0 0 10px #00ff00;
        }

        .best-box {
            font-size: 18px;
            color: #aaa;
        }

        #feedback-overlay {
            position: absolute;
            top: 45%;
            left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            pointer-events: none;
            opacity: 0;
            transition: opacity 0.2s;
            z-index: 30;
        }

        #feedback-text {
            font-size: 90px;
            font-weight: 600;
            color: white;
            text-shadow: 0 0 20px rgba(0,0,0,0.9);
            margin: 0;
            font-style: italic;
        }

        #feedback-sub {
            font-size: 30px;
            color: #ccc;
            background: rgba(0,0,0,0.8);
            padding: 5px 20px;
            border-radius: 4px;
        }

        .instructions {
            position: absolute;
            bottom: 20px;
            width: 100%;
            text-align: center;
            color: rgba(255,255,255,0.6);
            font-size: 26px;
            font-weight: bold;
            pointer-events: none;
            letter-spacing: 1px;
            text-shadow: 0 2px 4px #000;
        }

        .scanlines {
            position: absolute;
            top: 0; left: 0; right: 0; bottom: 0;
            background: linear-gradient(rgba(18, 16, 16, 0) 50%, rgba(0, 0, 0, 0.1) 50%);
            background-size: 100% 4px;
            pointer-events: none;
            z-index: 10;
        }

    </style>
</head>
<body>

    <div id="game-container">
        <canvas id="gameCanvas"></canvas>
        <div class="scanlines"></div>
        
        <div id="ui-layer">
            <div>
                <div class="stat-box">
                    Goals: <span id="score" class="score-big">0</span>
                </div>
                <div class="stat-box" style="border-left-color: #ff3333;">
                    Misses: <span id="misses" class="score-big" style="color: #ff3333;">0</span>
                </div>
            </div>
            
            <div class="streak-container">
                <div class="streak-box">STREAK: <span id="streak">0</span> 🔥</div>
                <div class="best-box">BEST: <span id="best">0</span> 🏆</div>
            </div>
        </div>

        <div id="feedback-overlay">
            <h1 id="feedback-text">GOAL!</h1>
            <div id="feedback-sub">Top Right Corner</div>
        </div>
        
        <div class="instructions">DRAW A LINE TO SHOOT</div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const uiScore = document.getElementById('score');
        const uiMisses = document.getElementById('misses');
        const uiStreak = document.getElementById('streak');
        const uiBest = document.getElementById('best');
        const uiFeedback = document.getElementById('feedback-overlay');
        const txtMain = document.getElementById('feedback-text');
        const txtSub = document.getElementById('feedback-sub');

        canvas.width = 1000;
        canvas.height = 700;

        // Added WINDING_UP state for the animation sequence
        const STATE = { IDLE: 0, DRAGGING: 1, WINDING_UP: 2, SHOOTING: 3, CELEBRATION: 4 };
        let gameState = STATE.IDLE;
        
        let stats = { goals: 0, misses: 0, streak: 0 };
        let bestStreak = localStorage.getItem('proPenaltyBest') || 0;
        uiBest.innerText = bestStreak;
        
        const groundY = 500;
        const goal = { x: 250, y: 220, w: 500, h: 280, topY: 220 };
        const ballStart = { x: 500, y: 600, radius: 20 };

        let ball = {
            x: ballStart.x, y: ballStart.y, z: 0,
            radius: ballStart.radius,
            rotation: 0,
            vx: 0, vy: 0, vz: 0 
        };

        let keeper = {
            x: 500, y: 500, width: 80, height: 160,
            targetX: 500, targetY: 500,
            diveState: 'idle'
        };

        let shooter = {
            x: 450, y: 580,
            legAngle: 0,
            shootState: 'idle'
        };

        let pathPoints = []; 
        let pathProgress = 0;
        const DEFAULT_POWER = 0.95; 
        let particles = [];

        function getPos(e) {
            const rect = canvas.getBoundingClientRect();
            const clientX = e.touches ? e.touches[0].clientX : e.clientX;
            const clientY = e.touches ? e.touches[0].clientY : e.clientY;
            const scaleX = canvas.width / rect.width;
            const scaleY = canvas.height / rect.height;
            return { x: (clientX - rect.left) * scaleX, y: (clientY - rect.top) * scaleY };
        }

        function startInput(e) {
            if (gameState !== STATE.IDLE) return;
            const pos = getPos(e);
            if (Math.hypot(pos.x - ball.x, pos.y - ball.y) < 200) {
                gameState = STATE.DRAGGING;
                pathPoints = [pos];
                // Shooter remains idle while drawing
            }
        }

        function moveInput(e) {
            if (gameState !== STATE.DRAGGING) return;
            e.preventDefault(); 
            const pos = getPos(e);
            pathPoints.push(pos);
        }

        function endInput(e) {
            if (gameState !== STATE.DRAGGING) return;
            
            if (pathPoints.length > 5) {
                // Drawing finished. Now start the windup animation.
                gameState = STATE.WINDING_UP;
                shooter.shootState = 'windup';
            } else {
                gameState = STATE.IDLE;
                shooter.shootState = 'idle';
            }
        }

        canvas.addEventListener('mousedown', startInput);
        canvas.addEventListener('mousemove', moveInput);
        window.addEventListener('mouseup', endInput);
        canvas.addEventListener('touchstart', startInput, {passive: false});
        canvas.addEventListener('touchmove', moveInput, {passive: false});
        window.addEventListener('touchend', endInput);

        // --- GAME LOGIC ---

        function decideKeeperAI(ballDestX, ballDestY) {
            let diveX, diveY, reactionTime;
            if (Math.random() < 0.4) {
                diveX = ballDestX + (Math.random() - 0.5) * 80;
            } else {
                diveX = goal.x + (Math.random() * goal.w);
            }
            if (ballDestY < 350) diveY = (Math.random() > 0.3) ? 300 : 450; 
            else diveY = 450;
            // Slight delay so he doesn't move before the ball gets kicked
            reactionTime = 200 + (Math.random() * 150);
            diveX = Math.max(goal.x - 20, Math.min(goal.x + goal.w + 20, diveX));
            setTimeout(() => {
                keeper.targetX = diveX; keeper.targetY = diveY; keeper.diveState = 'diving';
            }, reactionTime);
        }

        function spawnConfetti(x, y) {
            for(let i=0; i<60; i++) { 
                particles.push({
                    x: x, y: y,
                    vx: (Math.random() - 0.5) * 15,
                    vy: (Math.random() - 0.5) * 15,
                    life: 1.0,
                    color: `hsl(${Math.random()*360}, 100%, 50%)`
                });
            }
        }

        function updateParticles() {
            for(let i=particles.length-1; i>=0; i--) {
                let p = particles[i];
                p.x += p.vx; p.y += p.vy; p.vy += 0.5; p.life -= 0.02;
                if(p.life <= 0) particles.splice(i, 1);
            }
        }

        function update() {
            updateParticles();

            // --- Shooter Animation Sequence ---
            if (shooter.shootState === 'windup') {
                // 1. Wind up leg (happens in WINDING_UP state)
                shooter.legAngle -= 8; 
                if (shooter.legAngle < -70) {
                     shooter.legAngle = -70;
                     // Windup complete. Transition to kicking.
                     if (gameState === STATE.WINDING_UP) {
                         shooter.shootState = 'kicking';
                         // Start the shot logic now
                         gameState = STATE.SHOOTING;
                         pathProgress = 0;
                         let endPoint = pathPoints[pathPoints.length-1];
                         decideKeeperAI(endPoint.x, endPoint.y);
                     }
                }
            } else if (shooter.shootState === 'kicking') {
                // 2. Snap leg forward (happens at start of SHOOTING state)
                shooter.legAngle += 25;
                if (shooter.legAngle > 45) {
                     shooter.legAngle = 45;
                     // Reset leg after a short delay
                     setTimeout(() => { 
                         if(gameState === STATE.CELEBRATION || gameState === STATE.IDLE) {
                             shooter.shootState = 'idle';
                         }
                     }, 500);
                }
            } else {
                // Idle state return
                shooter.legAngle *= 0.9;
            }

            // --- Ball Physics ---
            if (gameState === STATE.SHOOTING) {
                let speed = 0.015 + (DEFAULT_POWER * 0.025);
                pathProgress += speed;
                if (pathProgress >= 1) { pathProgress = 1; checkResult(); }
                let floatIndex = pathProgress * (pathPoints.length - 1);
                let index = Math.floor(floatIndex);
                let nextIndex = Math.min(index + 1, pathPoints.length - 1);
                let t = floatIndex - index;
                let p1 = pathPoints[index]; let p2 = pathPoints[nextIndex];
                ball.x = p1.x + (p2.x - p1.x) * t;
                ball.y = p1.y + (p2.y - p1.y) * t;
                ball.z = pathProgress; 
                ball.rotation += 25;
                ball.vx = (p2.x - p1.x) * 0.5; ball.vy = (p2.y - p1.y) * 0.5;
                keeper.x += (keeper.targetX - keeper.x) * 0.1;
                keeper.y += (keeper.targetY - keeper.y) * 0.1;
                ball.radius = 20 * (1 - (ball.z * 0.4));
            } 
            else if (gameState === STATE.CELEBRATION) {
                if (txtMain.innerText !== "GOAL!") {
                    ball.vy += 0.8; ball.vx *= 0.98; ball.vz *= 0.98;
                    ball.x += ball.vx; ball.y += ball.vy;
                    if (ball.y > groundY + 10) { ball.y = groundY + 10; ball.vy *= -0.6; }
                    ball.rotation += ball.vx; ball.radius = 20 * (1 - (ball.z * 0.4));
                }
            }
        }

        function checkResult() {
            gameState = STATE.CELEBRATION;
            const bx = ball.x; const by = ball.y; const bRad = 12; 
            const inLeft = bx + bRad > goal.x; const inRight = bx - bRad < goal.x + goal.w;
            const inTop = by + bRad > goal.topY; const inGround = by - bRad < groundY;
            const isGoalXY = inLeft && inRight && inTop && inGround;
            let distX = Math.abs(bx - keeper.x); let distY = Math.abs(by - (keeper.y - 60)); 
            let keeperSave = (distX < 60 && distY < 90);
            let result = "", reason = "", isGoal = false;

            if (keeperSave) {
                result = "SAVED"; reason = "Great Save!";
                stats.misses++; stats.streak = 0;
                ball.vz = -0.5; ball.vy = -5 - Math.random() * 5; ball.vx = (bx - keeper.x) * 0.5;
            } 
            else if (isGoalXY) {
                result = "GOAL!"; isGoal = true;
                stats.goals++; stats.streak++;
                if (stats.streak > bestStreak) {
                    bestStreak = stats.streak; localStorage.setItem('proPenaltyBest', bestStreak); uiBest.innerText = bestStreak;
                }
                spawnConfetti(bx, by); 
                let distPost = Math.min(Math.abs(bx - goal.x), Math.abs(bx - (goal.x+goal.w)));
                let distBar = Math.abs(by - goal.topY);
                if (distPost < 40 && distBar < 40) reason = "TOP BIN!";
                else if (distPost < 30) reason = "Off the Post!";
                else if (distBar < 30) reason = "Bar Down!";
                else reason = "Beautiful Finish";
            } 
            else {
                result = "MISS"; stats.misses++; stats.streak = 0;
                if (by < goal.topY + 15 && by > goal.topY - 15 && inLeft && inRight) {
                    reason = "Crossbar!"; ball.vy = -8 - Math.random()*5; ball.vz = -0.2;
                } else if ((Math.abs(bx - goal.x) < 15 || Math.abs(bx - (goal.x+goal.w)) < 15) && inTop) {
                    reason = "Post!"; ball.vx = -ball.vx * 0.8; ball.vz = -0.2;
                } else reason = "Wide";
            }
            uiScore.innerText = stats.goals; uiMisses.innerText = stats.misses; uiStreak.innerText = stats.streak;
            showFeedback(result, reason, isGoal);
            setTimeout(resetGame, 2000);
        }

        function resetGame() {
            uiFeedback.style.opacity = 0;
            ball = { ...ballStart, z:0, rotation:0, radius:20, vx:0, vy:0, vz:0 };
            keeper.x = 500; keeper.y = 500; keeper.diveState = 'idle'; keeper.targetX = 500;
            shooter.legAngle = 0; shooter.shootState = 'idle'; 
            pathPoints = []; gameState = STATE.IDLE;
        }

        function showFeedback(main, sub, isGoal) {
            txtMain.innerText = main;
            txtMain.style.color = isGoal ? '#00ff00' : '#ff3333';
            txtSub.innerText = sub;
            uiFeedback.style.opacity = 1;
        }

        // --- DRAWING ---

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            drawSky(); drawStadium(); drawPitch(); drawGoal(); drawShadows();
            drawKeeper();
            drawShooter(); 
            if (pathPoints.length > 0 && gameState === STATE.DRAGGING) drawSwipePath();
            drawBall();
            drawParticles();
        }

        function drawShooter() {
            ctx.save();
            ctx.translate(shooter.x, shooter.y);

            const jerseyColor = "#e63946";
            const shortsColor = "#1d3557";
            const socksColor = "#e63946";
            const skinColor = "#d4a373";

            // Torso
            ctx.fillStyle = jerseyColor; ctx.fillRect(-25, -90, 50, 55);
            
            // Jersey Text
            ctx.fillStyle = "white"; ctx.textAlign = "center";
            ctx.font = "bold 14px Teko"; ctx.fillText("RONALDO", 0, -75);
            ctx.font = "bold 28px Teko"; ctx.fillText("7", 0, -50);

            // Shorts
            ctx.fillStyle = shortsColor; ctx.fillRect(-22, -35, 44, 35);

            // Stationary Leg (Left)
            ctx.fillStyle = shortsColor; ctx.fillRect(-20, 0, 15, 25);
            ctx.fillStyle = socksColor; ctx.fillRect(-20, 25, 15, 30);
            ctx.fillStyle = "#111"; ctx.fillRect(-22, 55, 17, 10);

            // Kicking Leg (Right) - Rotates
            ctx.save();
            ctx.translate(12, 0); 
            ctx.rotate(shooter.legAngle * Math.PI/180);
            ctx.fillStyle = shortsColor; ctx.fillRect(-7, 0, 15, 25);
            ctx.fillStyle = socksColor; ctx.fillRect(-7, 25, 15, 30);
            ctx.fillStyle = "#111"; ctx.fillRect(-5, 55, 17, 10); 
            ctx.restore();

            // Arms (Articulated)
            ctx.fillStyle = jerseyColor;
            // Left Arm (Upper & Lower)
            ctx.save(); ctx.translate(-30, -85); ctx.rotate(Math.PI/12);
            ctx.fillRect(-5, 0, 12, 25); // Upper
            ctx.translate(0, 25); ctx.rotate(-Math.PI/6);
            ctx.fillStyle = skinColor; ctx.fillRect(-4, 0, 10, 25); // Lower/Hand
            ctx.restore();

            // Right Arm (Upper & Lower) - balance leg kick
            ctx.save(); ctx.translate(30, -85); 
            // Rotate arm based on leg kick for balance
            let armBal = -shooter.legAngle * 0.5;
            ctx.rotate(-Math.PI/12 + armBal*Math.PI/180);
            ctx.fillStyle = jerseyColor; ctx.fillRect(-7, 0, 12, 25); // Upper
            ctx.translate(0, 25); ctx.rotate(Math.PI/8);
            ctx.fillStyle = skinColor; ctx.fillRect(-6, 0, 10, 25); // Lower/Hand
            ctx.restore();
            
            // Head
            ctx.fillStyle = skinColor; ctx.beginPath(); ctx.arc(0, -105, 15, 0, Math.PI*2); ctx.fill();
            
            // Styled Hair (Spiky)
            ctx.fillStyle = "#111"; 
            ctx.beginPath();
            ctx.moveTo(-15, -105);
            ctx.lineTo(-18, -115); ctx.lineTo(-10, -122);
            ctx.lineTo(0, -125); ctx.lineTo(10, -122);
            ctx.lineTo(18, -115); ctx.lineTo(15, -105);
            ctx.fill();

            ctx.restore();
        }

        function drawParticles() {
            for(let p of particles) {
                ctx.fillStyle = p.color; ctx.globalAlpha = p.life;
                ctx.beginPath(); ctx.rect(p.x, p.y, 8, 8); ctx.fill();
            }
            ctx.globalAlpha = 1;
        }

        function drawSwipePath() {
            ctx.save(); ctx.lineCap = "round"; ctx.lineJoin = "round";
            ctx.shadowBlur = 10; ctx.shadowColor = "#00ffff";
            ctx.strokeStyle = "rgba(0, 255, 255, 0.8)"; ctx.lineWidth = 6;
            ctx.beginPath(); ctx.moveTo(pathPoints[0].x, pathPoints[0].y);
            for (let i = 1; i < pathPoints.length; i++) ctx.lineTo(pathPoints[i].x, pathPoints[i].y);
            ctx.stroke(); ctx.restore();
        }

        function drawSky() {
            let grad = ctx.createLinearGradient(0, 0, 0, 300);
            grad.addColorStop(0, "#1a2a6c"); grad.addColorStop(1, "#b21f1f");
            ctx.fillStyle = grad; ctx.fillRect(0, 0, canvas.width, groundY);
        }

        function drawStadium() {
            ctx.fillStyle = "#111"; ctx.fillRect(0, 150, canvas.width, 100);
            for(let i=100; i<1000; i+=200) {
                let grad = ctx.createRadialGradient(i, 50, 0, i, 50, 60);
                grad.addColorStop(0, "rgba(255,255,255,0.8)"); grad.addColorStop(1, "rgba(255,255,255,0)");
                ctx.fillStyle = grad; ctx.beginPath(); ctx.arc(i, 50, 60, 0, Math.PI*2); ctx.fill();
            }
        }

        function drawPitch() {
            let grad = ctx.createLinearGradient(0, groundY, 0, canvas.height);
            grad.addColorStop(0, "#2c5e1a"); grad.addColorStop(1, "#3f8c25");
            ctx.fillStyle = grad; ctx.fillRect(0, groundY-50, canvas.width, canvas.height);
            ctx.strokeStyle = "rgba(255,255,255,0.3)"; ctx.lineWidth = 2;
            ctx.beginPath(); ctx.moveTo(200, groundY); ctx.lineTo(0, canvas.height);
            ctx.moveTo(800, groundY); ctx.lineTo(1000, canvas.height);
            ctx.moveTo(250, groundY); ctx.lineTo(250, groundY+50);
            ctx.lineTo(750, groundY+50); ctx.lineTo(750, groundY); ctx.stroke();
        }

        function drawGoal() {
            ctx.save(); ctx.strokeStyle = "#e0e0e0"; ctx.lineWidth = 6;
            ctx.beginPath(); ctx.moveTo(goal.x, groundY); ctx.lineTo(goal.x, goal.topY);
            ctx.lineTo(goal.x + goal.w, goal.topY); ctx.lineTo(goal.x + goal.w, groundY); ctx.stroke();
            const bx = goal.x + 20, bw = goal.w - 40, by = goal.topY + 10;
            ctx.lineWidth = 4; ctx.beginPath();
            ctx.moveTo(bx, groundY); ctx.lineTo(bx, by); ctx.moveTo(bx + bw, groundY); ctx.lineTo(bx + bw, by);
            ctx.moveTo(bx, by); ctx.lineTo(bx + bw, by); ctx.moveTo(goal.x, goal.topY); ctx.lineTo(bx, by);
            ctx.moveTo(goal.x + goal.w, goal.topY); ctx.lineTo(bx + bw, by); ctx.stroke();
            ctx.strokeStyle = "rgba(255,255,255,0.2)"; ctx.lineWidth = 1;
            const hr = 12, hh = Math.sqrt(3) * hr;
            ctx.beginPath();
            for (let y = goal.topY; y <= groundY; y += hh) {
                for (let x = goal.x; x <= goal.x + goal.w; x += hr * 3) {
                    if (x + hr < goal.x + goal.w) {
                        ctx.moveTo(x + hr, y); ctx.lineTo(x + hr * 2, y);
                        ctx.lineTo(x + hr * 2.5, y + hh / 2); ctx.lineTo(x + hr * 2, y + hh);
                        ctx.lineTo(x + hr, y + hh); ctx.lineTo(x + hr * 0.5, y + hh / 2); ctx.closePath();
                    }
                }
            }
            ctx.stroke(); ctx.restore();
        }

        function drawShadows() {
            ctx.fillStyle = "rgba(0,0,0,0.4)";
            let ss = Math.max(0.1, 1 - ball.z);
            let sy = groundY + (ball.y - groundY)*0.1 + (200 * ball.z);
            if (ball.y < groundY - 50 && ball.z < 0.8) ctx.globalAlpha = 0.5;
            ctx.beginPath(); ctx.ellipse(ball.x, sy, 15*ss, 5*ss, 0, 0, Math.PI*2); ctx.fill();
            ctx.globalAlpha = 1;
            ctx.beginPath(); ctx.ellipse(keeper.x, 500, 40, 10, 0, 0, Math.PI*2); ctx.fill();
            ctx.beginPath(); ctx.ellipse(shooter.x, shooter.y+60, 35, 8, 0, 0, Math.PI*2); ctx.fill();
        }

        function drawKeeper() {
            ctx.save(); ctx.translate(keeper.x, keeper.y);
            ctx.rotate((keeper.x - 500) * 0.05 * Math.PI/180);
            ctx.fillStyle = "#ffc107"; ctx.fillRect(-25, -100, 50, 60);
            
            // Keeper Number
            ctx.fillStyle = "black"; ctx.textAlign = "center";
            ctx.font = "bold 30px Teko"; ctx.fillText("1", 0, -60);

            ctx.fillStyle = "#111"; ctx.fillRect(-22, -40, 44, 40);
            ctx.fillRect(-20, 0, 15, 30); ctx.fillRect(5, 0, 15, 30);
            ctx.fillStyle = "#ffc107";
            if (keeper.diveState === 'diving') {
                ctx.fillRect(-55, -90, 30, 15); ctx.fillRect(25, -90, 30, 15);
                ctx.fillStyle = "#111"; ctx.fillRect(-65, -95, 10, 25); ctx.fillRect(55, -95, 10, 25);
            } else {
                ctx.fillRect(-40, -95, 15, 50); ctx.fillRect(25, -95, 15, 50);
                ctx.fillStyle = "#111"; ctx.fillRect(-45, -55, 10, 15); ctx.fillRect(35, -55, 10, 15);
            }
            ctx.fillStyle = "#7f4f24"; ctx.beginPath(); ctx.arc(0, -115, 15, 0, Math.PI*2); ctx.fill();
            ctx.fillStyle = "#111"; ctx.beginPath(); ctx.arc(0, -120, 16, Math.PI, 0); ctx.fill();
            ctx.restore();
        }

        function drawBall() {
            ctx.save(); ctx.translate(ball.x, ball.y);
            ctx.rotate(ball.rotation * Math.PI / 180);
            ctx.beginPath(); ctx.arc(0, 0, ball.radius, 0, Math.PI*2);
            let g = ctx.createRadialGradient(-5, -5, 2, 0, 0, ball.radius);
            g.addColorStop(0, "#fff"); g.addColorStop(1, "#e0e0e0"); ctx.fillStyle = g; ctx.fill();
            ctx.fillStyle = "#0056b3"; 
            ctx.beginPath(); ctx.moveTo(0, -ball.radius/2.5); ctx.lineTo(ball.radius/2.5, -ball.radius/6);
            ctx.lineTo(ball.radius/3.5, ball.radius/3); ctx.lineTo(-ball.radius/3.5, ball.radius/3);
            ctx.lineTo(-ball.radius/2.5, -ball.radius/6); ctx.fill(); ctx.restore();
        }

        function loop() { update(); draw(); requestAnimationFrame(loop); }
        loop();

    </script>
</body>
</html>
