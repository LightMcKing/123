<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Space Dodger Extreme</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
            touch-action: none;
        }
        body {
            margin: 0;
            padding: 0;
            background-color: #050515;
            color: white;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            overflow: hidden;
        }
        h1 {
            margin: 5px 0;
            font-size: 22px;
            letter-spacing: 3px;
            text-shadow: 0 0 10px #00ffff;
            color: #00ffff;
        }
        #gameCanvas {
            border: 3px solid #00ffff;
            background-color: #020208;
            box-shadow: 0 0 20px #00ffff;
            max-width: 95vw;
            max-height: 70vh;
            border-radius: 12px;
        }
        .instructions {
            margin: 10px 0;
            font-size: 13px;
            color: #88aaee;
            text-align: center;
            max-width: 90%;
            line-height: 1.4;
        }
    </style>
</head>
<body>

    <h1>GOONER SMASHER</h1>
    <canvas id="gameCanvas" width="360" height="520"></canvas>
    <div class="instructions">Left Half: Drag to Move Across Whole Screen | Right Half: Tap to Shoot Lasers<br>Tap when defeated to restart.</div>

    <script>
        const canvas = document.getElementById("gameCanvas");
        const ctx = canvas.getContext("2d");

        let gameOver = false;
        let score = 0;
        let speedMultiplier = 1;

        // Player Space Ship
        const player = {
            x: canvas.width / 2 - 16,
            y: canvas.height - 70,
            width: 32,
            height: 32,
            color: "#00ffff"
        };

        // Projectile & Asteroid Tracking Arrays
        const obstacles = [];
        const lasers = [];
        
        const obstacleSpawnRate = 30; 
        let spawnTimer = 0;

        // --- MULTI-TOUCH INPUT SYSTEMS (FIXED FOR FULL TRACKING) ---
        function processTouches(e) {
            e.preventDefault();
            if (gameOver) return;

            const rect = canvas.getBoundingClientRect();
            
            // Loop through all active fingers on screen
            for (let i = 0; i < e.touches.length; i++) {
                const touch = e.touches[i];
                const canvasX = ((touch.clientX - rect.left) / rect.width) * canvas.width;

                // LEFT HALF OF SCREEN: Moves ship across the ENTIRE width of the gameboard
                if (canvasX < canvas.width / 2) {
                    player.x = (canvasX * 2) - player.width / 2;
                    
                    // Prevent flying out of game boundaries
                    if (player.x < 0) player.x = 0;
                    if (player.x > canvas.width - player.width) player.x = canvas.width - player.width;
                }
            }
        }

        // Tap Right side to trigger Weapon Fire
        canvas.addEventListener("touchstart", (e) => {
            if (gameOver) {
                resetGame();
                return;
            }

            const rect = canvas.getBoundingClientRect();
            // Look at the newest finger down
            const newestTouch = e.changedTouches[0];
            const canvasX = ((newestTouch.clientX - rect.left) / rect.width) * canvas.width;

            // RIGHT HALF OF SCREEN: Weapon System
            if (canvasX >= canvas.width / 2) {
                fireLaser();
            } else {
                processTouches(e);
            }
        }, { passive: false });

        canvas.addEventListener("touchmove", processTouches, { passive: false });

        // Keyboard backup systems for testing
        window.addEventListener("keydown", (e) => {
            if (e.code === "Space") {
                if (gameOver) resetGame();
                else fireLaser();
            }
            if (e.code === "ArrowLeft" && player.x > 0) player.x -= 20;
            if (e.code === "ArrowRight" && player.x < canvas.width - player.width) player.x += 20;
        });

        function fireLaser() {
            lasers.push({
                x: player.x + player.width / 2 - 2, // Centered on ship tip
                y: player.y,
                width: 4,
                height: 15,
                speed: 11,
                color: "#ff00ff"
            });
        }

        function resetGame() {
            obstacles.length = 0;
            lasers.length = 0;
            score = 0;
            speedMultiplier = 1;
            player.x = canvas.width / 2 - 16;
            gameOver = false;
            animate();
        }

        function spawnObstacle() {
            const roll = Math.random();
            let size, hp, speedOffset, color, points, name, isFinalBoss = false;

            // 6-Tier Custom Roster Allocation
            if (roll < 0.40) {
                size = 18; hp = 1; speedOffset = 3.6; color = "#ff5555"; points = 1; name = "mass the gay gooner";
            } else if (roll < 0.70) {
                size = 26; hp = 2; speedOffset = 2.6; color = "#ffaa00"; points = 3; name = "George the secret gooner";
            } else if (roll < 0.86) {
                size = 32; hp = 3; speedOffset = 2.0; color = "#ffff33"; points = 5; name = "West the og gooner";
            } else if (roll < 0.94) {
                size = 40; hp = 5; speedOffset = 1.4; color = "#cc33ff"; points = 12; name = "phantom the serial gooner";
            } else if (roll < 0.98) {
                size = 48; hp = 10; speedOffset = 1.0; color = "#33ff33"; points = 25; name = "Aura farmer the ultimate gooner";
            } else {
                size = 56; hp = 25; speedOffset = 0.6; color = "#ffffff"; points = 100; name = "Jassy the legendary gooner";
                isFinalBoss = true;
            }

            obstacles.push({
                x: Math.random() * (canvas.width - size),
                y: -size,
                width: size,
                height: size,
                maxHp: hp,
                currentHp: hp,
                speed: speedOffset * speedMultiplier,
                color: color,
                points: points,
                name: name,
                isFinalBoss: isFinalBoss,
                flashFrames: 0,
                hue: 0
            });
        }

        // Box collision helper standard formula
        function checkCollision(r1, r2) {
            return r1.x < r2.x + r2.width &&
                   r1.x + r1.width > r2.x &&
                   r1.y < r2.y + r2.height &&
                   r1.y + r1.height > r2.y;
        }

        // Main Rendering Loop
        function animate() {
            if (gameOver) {
                ctx.fillStyle = "rgba(2, 2, 8, 0.95)";
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                
                ctx.fillStyle = "#ff3366";
                ctx.font = "bold 32px sans-serif";
                ctx.textAlign = "center";
                ctx.fillText("MISSION FAILED", canvas.width / 2, canvas.height / 2 - 20);
                
                ctx.fillStyle = "white";
                ctx.font = "18px sans-serif";
                ctx.fillText(`Final Score: ${score}`, canvas.width / 2, canvas.height / 2 + 20);
                
                ctx.fillStyle = "#00ffff";
                ctx.font = "14px sans-serif";
                ctx.fillText("TAP SCREEN TO REDEPLOY", canvas.width / 2, canvas.height / 2 + 70);
                return;
            }

            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // 1. Move & Render Active Lasers
            for (let i = lasers.length - 1; i >= 0; i--) {
                let l = lasers[i];
                l.y -= l.speed;

                ctx.fillStyle = l.color;
                ctx.fillRect(l.x, l.y, l.width, l.height);

                if (l.y < -20) {
                    lasers.splice(i, 1);
                }
            }

            // 2. Render Player Ship (Cyan Neon Jet)
            ctx.fillStyle = player.color;
            ctx.shadowBlur = 12;
            ctx.shadowColor = player.color;
            ctx.beginPath();
            ctx.moveTo(player.x + player.width / 2, player.y);
            ctx.lineTo(player.x, player.y + player.height);
            ctx.lineTo(player.x + player.width, player.y + player.height);
            ctx.closePath();
            ctx.fill();
            ctx.shadowBlur = 0;

            // 3. Spawning & Handling Asteroids
            spawnTimer++;
            if (spawnTimer >= obstacleSpawnRate) {
                spawnObstacle();
                spawnTimer = 0;
            }

            for (let i = obstacles.length - 1; i >= 0; i--) {
                let obs = obstacles[i];
                obs.y += obs.speed;

                // Cycling boss color
                if (obs.isFinalBoss) {
                    obs.hue = (obs.hue + 5) % 360;
                    obs.color = `hsl(${obs.hue}, 100%, 60%)`;
                }

                if (obs.flashFrames > 0) {
                    ctx.fillStyle = "#ffffff";
                    obs.flashFrames--;
                } else {
                    ctx.fillStyle = obs.color;
                }
                ctx.fillRect(obs.x, obs.y, obs.width, obs.height);

                // Draw Text Titles Above Cubes
                ctx.fillStyle = obs.isFinalBoss ? "#ffffff" : obs.color;
                ctx.font = "bold 11px sans-serif";
                ctx.textAlign = "center";
