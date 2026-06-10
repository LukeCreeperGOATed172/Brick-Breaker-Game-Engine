<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Brick Engine v20.7</title>
    <style>
        body {
            margin: 0;
            background: #000;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
            font-family: monospace;
        }
        canvas {
            background: #111;
            border: 4px solid #333;
            box-shadow: 0 0 20px rgba(0, 255, 204, 0.2);
        }
    </style>
</head>
<body>

    <canvas id="game" width="800" height="500"></canvas>

    <script>
    class Input {
        static keys = {};
        static pressed = {};
        static init() {
            window.addEventListener("keydown", (e) => {
                if (!Input.keys[e.code]) Input.pressed[e.code] = true;
                Input.keys[e.code] = true;
            });
            window.addEventListener("keyup", (e) => Input.keys[e.code] = false);
        }
        static update() { Input.pressed = {}; }
        static isKeyDown(code) { return !!Input.keys[code]; }
        static isKeyPressed(code) { return !!Input.pressed[code]; }
    }
    Input.init();

    class AudioEngine {
        static ctx = null;
        static init() {
            if (!AudioEngine.ctx) AudioEngine.ctx = new (window.AudioContext || window.webkitAudioContext)();
        }
        static playTone(freq, type, duration, vol = 0.1) {
            try {
                AudioEngine.init();
                if (AudioEngine.ctx.state === 'suspended') AudioEngine.ctx.resume();
                const osc = AudioEngine.ctx.createOscillator();
                const gain = AudioEngine.ctx.createGain();
                osc.type = type;
                osc.frequency.setValueAtTime(freq, AudioEngine.ctx.currentTime);
                gain.gain.setValueAtTime(vol, AudioEngine.ctx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.00001, AudioEngine.ctx.currentTime + duration);
                osc.connect(gain);
                gain.connect(AudioEngine.ctx.destination);
                osc.start();
                osc.stop(AudioEngine.ctx.currentTime + duration);
            } catch (e) {}
        }
        static playHit() { AudioEngine.playTone(180, "triangle", 0.08, 0.15); }
        static playScore() { AudioEngine.playTone(520, "square", 0.06, 0.04); }
        static playPowerUp() {
            AudioEngine.playTone(330, "sine", 0.1, 0.08);
            setTimeout(() => AudioEngine.playTone(660, "sine", 0.15, 0.08), 80);
        }
        static playLose() {
            AudioEngine.playTone(220, "sawtooth", 0.15, 0.08);
            setTimeout(() => AudioEngine.playTone(110, "sawtooth", 0.25, 0.08), 120);
        }
    }

    class SceneManager {
        constructor(game) { this.game = game; this.current = null; }
        change(sceneInstance) {
            this.current = sceneInstance;
            if (this.current && typeof this.current.init === "function") this.current.init();
        }
        update() { if (this.current && typeof this.current.update === "function") this.current.update(); }
        render() { if (this.current && typeof this.current.render === "function") this.current.render(this.game); }
    }

    class Game {
        constructor(canvasId) {
            this.canvas = document.getElementById(canvasId);
            this.ctx = this.canvas.getContext("2d");
            this.scene = new SceneManager(this);
        }
        update() { this.scene.update(); Input.update(); }
        render() { this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height); this.scene.render(); }
    }

    const Physics = {
        ballPaddle: function(ball, paddle) {
            if (ball.x + ball.radius > paddle.x && ball.x - ball.radius < paddle.x + paddle.width &&
                ball.y + ball.radius > paddle.y && ball.y - ball.radius < paddle.y + paddle.height) {
                AudioEngine.playHit();
                ball.y = paddle.y - ball.radius;
                ball.vy = -Math.abs(ball.vy);
                let hitPos = (ball.x - paddle.x) / paddle.width;
                ball.vx = 8 * (hitPos - 0.5);
            }
        },
        ballBrick: function(ball, brick, scene) {
            if (!brick.active) return;
            let closestX = Math.max(brick.x, Math.min(ball.x, brick.x + brick.width));
            let closestY = Math.max(brick.y, Math.min(ball.y, brick.y + brick.height));
            let distanceX = ball.x - closestX;
            let distanceY = ball.y - closestY;
            let distanceSquared = (distanceX * distanceX) + (distanceY * distanceY);
            if (distanceSquared < (ball.radius * ball.radius)) {
                let overlapX = ball.radius - Math.abs(distanceX);
                let overlapY = ball.radius - Math.abs(distanceY);
                if (overlapX < overlapY) {
                    ball.vx = distanceX > 0 ? Math.abs(ball.vx) : -Math.abs(ball.vx);
                    ball.x += distanceX > 0 ? overlapX : -overlapX;
                } else {
                    ball.vy = distanceY > 0 ? Math.abs(ball.vy) : -Math.abs(ball.vy);
                    ball.y += distanceY > 0 ? overlapY : -overlapY;
                }
                brick.active = false;
                scene.score += 100;
                AudioEngine.playScore();
                if (Math.random() < 0.20) {
                    const types = ["LIFE", "WIDE", "SLOW"];
                    const choice = types[Math.floor(Math.random() * types.length)];
                    scene.powerUps.push(new PowerUp(brick.x + brick.width / 2, brick.y, choice));
                }
            }
        }
    };

    class Paddle {
        constructor() { this.width = 100; this.height = 15; this.x = 350; this.y = 450; this.speed = 8; }
        update() {
            if (Input.isKeyDown("ArrowLeft") || Input.isKeyDown("KeyA")) this.x -= this.speed;
            if (Input.isKeyDown("ArrowRight") || Input.isKeyDown("KeyD")) this.x += this.speed;
            if (this.x < 0) this.x = 0;
            if (this.x > 800 - this.width) this.x = 800 - this.width;
        }
        draw(ctx) { ctx.fillStyle = "#00ffcc"; ctx.fillRect(this.x, this.y, this.width, this.height); }
    }

    class Ball {
        constructor() { this.reset(); }
        reset() {
            this.x = 400; this.y = 300; this.radius = 8;
            this.vx = 3 * (Math.random() > 0.5 ? 1 : -1); this.vy = -4;
        }
        update() {
            this.x += this.vx; this.y += this.vy;
            if (this.x - this.radius < 0 || this.x + this.radius > 800) { this.vx = -this.vx; AudioEngine.playHit(); }
            if (this.y - this.radius < 0) { this.vy = -this.vy; AudioEngine.playHit(); }
        }
        draw(ctx) {
            ctx.beginPath(); ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = "#ffffff"; ctx.fill(); ctx.closePath();
        }
    }

    class PowerUp {
        constructor(x, y, type) { this.x = x; this.y = y; this.type = type; this.radius = 12; this.speed = 2.5; }
        update() { this.y += this.speed; }
        draw(ctx) {
            ctx.beginPath(); ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            if (this.type === "LIFE") ctx.fillStyle = "#ff3366";
            if (this.type === "WIDE") ctx.fillStyle = "#33ccff";
            if (this.type === "SLOW") ctx.fillStyle = "#ffff33";
            ctx.fill(); ctx.closePath();
            ctx.fillStyle = "black"; ctx.font = "bold 10px sans-serif"; ctx.fillText(this.type[0], this.x - 3, this.y + 4);
        }
    }

    function createLevel(levelNum) {
        const bricks = []; const rows = Math.min(2 + levelNum, 7); const cols = 10;
        const brickWidth = 70; const brickHeight = 18; const padding = 5; const offsetTop = 60; const offsetLeft = 35;
        const colors = ["#ff3333", "#ff9933", "#ffff33", "#33cc33", "#3333ff", "#9933ff", "#ff33cc"];
        for (let r = 0; r < rows; r++) {
            for (let c = 0; c < cols; c++) {
                if (levelNum === 2 && (r + c) % 4 === 0) continue;
                if (levelNum >= 3 && r % 2 === 0 && c % 3 === 0) continue;
                bricks.push({
                    x: c * (brickWidth + padding) + offsetLeft, y: r * (brickHeight + padding) + offsetTop,
                    width: brickWidth, height: brickHeight, active: true, color: colors[r % colors.length],
                    draw: function(ctx) {
                        if (!this.active) return;
                        ctx.fillStyle = this.color; ctx.fillRect(this.x, this.y, this.width, this.height);
                        ctx.strokeStyle = "#111"; ctx.strokeRect(this.x, this.y, this.width, this.height);
                    }
                });
            }
        }
        return bricks;
    }

    class MenuScene {
        constructor(game) { this.game = game; this.startLevel = 1; }
        init() {
            this.highScore = 0; this.maxLevelReached = 1; this.startLevel = 1;
            try {
                const savedScore = localStorage.getItem("brick_highScore");
                const savedLevel = localStorage.getItem("brick_maxLevel");
                if (savedScore) this.highScore = parseInt(savedScore) || 0;
                if (savedLevel) this.maxLevelReached = parseInt(savedLevel) || 1;
            } catch (e) {}
        }
        update() {
            if (Input.isKeyPressed("ArrowUp") || Input.isKeyPressed("KeyW")) {
                if (this.startLevel < this.maxLevelReached) { this.startLevel++; AudioEngine.playHit(); }
            }
            if (Input.isKeyPressed("ArrowDown") || Input.isKeyPressed("KeyS")) {
                if (this.startLevel > 1) { this.startLevel--; AudioEngine.playHit(); }
            }
            if (Input.isKeyDown("Space")) { AudioEngine.playPowerUp(); this.game.scene.change(new GameScene(this.game, this.startLevel)); }
        }
        render(game) {
            const ctx = game.ctx; ctx.fillStyle = "white"; ctx.font = "28px monospace"; ctx.fillText("BRICK ENGINE v20.7", 250, 140);
            ctx.font = "16px monospace"; ctx.fillStyle = "#00ffcc"; ctx.fillText("PRESS SPACE TO PLAY", 295, 220);
            ctx.fillStyle = "#aaa"; ctx.font = "14px monospace";
            ctx.fillText("ALL-TIME HIGH SCORE: " + this.highScore, 280, 280);
            ctx.fillText("HIGHEST LEVEL UNLOCKED: Level " + this.maxLevelReached, 270, 310);
            ctx.fillStyle = "#ffff33"; ctx.fillText("START AT LEVEL: < Level " + this.startLevel + " >", 285, 360);
            ctx.fillStyle = "#666"; ctx.font = "12px monospace"; ctx.fillText("(Use UP/DOWN arrows to select an unlocked level)", 215, 385);
        }
    }

    class GameScene {
        constructor(game, selectedStartLevel = 1) { this.game = game; this.initialLevelSelection = selectedStartLevel; }
        init() {
            this.score = 0; this.lives = 3; this.level = this.initialLevelSelection; this.state = "playing";
            this.paddle = new Paddle(); this.ball = new Ball(); this.bricks = createLevel(this.level); this.powerUps = [];
            this.ball.vx *= (1 + (this.level - 1) * 0.1); this.ball.vy *= (1 + (this.level - 1) * 0.1);
        }
        saveProgress() {
            try {
                let currentHighScore = parseInt(localStorage.getItem("brick_highScore")) || 0;
                let currentMaxLevel = parseInt(localStorage.getItem("brick_maxLevel")) || 1;
                if (this.score > currentHighScore) localStorage.setItem("brick_highScore", this.score);
                if (this.level > currentMaxLevel) localStorage.setItem("brick_maxLevel", this.level);
            } catch (e) {}
        }
        update() {
            if (Input.isKeyPressed("Escape")) this.state = this.state === "playing" ? "paused" : "playing";
            if (Input.isKeyDown("KeyR")) { this.init(); return; }
            if (this.state === "gameover" && Input.isKeyDown("KeyM")) { this.game.scene.change(new MenuScene(this.game)); return; }
            if (this.state !== "playing") return;

            this.paddle.update(); this.ball.update();
            Physics.ballPaddle(this.ball, this.paddle);

            let levelClear = true;
            for (const b of this.bricks) { if (b.active) { levelClear = false; Physics.ballBrick(this.ball, b, this); } }

            if (levelClear) {
                this.level++; this.saveProgress(); AudioEngine.playPowerUp(); this.bricks = createLevel(this.level); this.ball.reset();
                this.ball.vx *= (1 + (this.level - 1) * 0.05); this.ball.vy *= (1 + (this.level - 1) * 0.05);
            }

            for (let i = this.powerUps.length - 1; i >= 0; i--) {
                let p = this.powerUps[i]; p.update();
                if (p.y + p.radius > this.paddle.y && p.x > this.paddle.x && p.x < this.paddle.x + this.paddle.width) {
                    AudioEngine.playPowerUp();
                    if (p.type === "LIFE") this.lives++;
                    if (p.type === "WIDE") { this.paddle.width = 160; setTimeout(() => this.paddle.width = 100, 8000); }
                    if (p.type === "SLOW") { this.ball.vx *= 0.7; this.ball.vy *= 0.7; }
                    this.powerUps.splice(i, 1); continue;
                }
                if (p.y > 520) this.powerUps.splice(i, 1);
            }

            if (this.ball.y > 500) {
                this.lives--; AudioEngine.playLose();
                if (this.lives <= 0) { this.state = "gameover"; this.saveProgress(); } else { this.ball.reset(); }
            }
        }
        render(game) {
            const ctx = game.ctx; this.paddle.draw(ctx); this.ball.draw(ctx);
            for (const b of this.bricks) b.draw(ctx); for (const p of this.powerUps) p.draw(ctx);
            ctx.fillStyle = "white"; ctx.font = "14px monospace";
            ctx.fillText("SCORE: " + this.score, 20, 25); ctx.fillText("LIVES: " + this.lives, 370, 25); ctx.fillText("LEVEL: " + this.level, 700, 25);

            if (this.state === "paused") {
                ctx.fillStyle = "rgba(0,0,0,0.6)"; ctx.fillRect(0, 0, 800, 500); ctx.fillStyle = "#ffff33"; ctx.font = "30px monospace";
                ctx.fillText("PAUSED", 350, 220); ctx.font = "14px monospace"; ctx.fillStyle = "white"; ctx.fillText("PRESS ESC TO RESUME", 325, 260);
            }
            if (this.state === "gameover") {
                ctx.fillStyle = "rgba(0,0,0,0.85)"; ctx.fillRect(0, 0, 800, 500); ctx.fillStyle = "#ff3333"; ctx.font = "32px monospace";
                ctx.fillText("GAME OVER", 315, 190); ctx.font = "16px monospace"; ctx.fillStyle = "white";
                ctx.fillText("FINAL SCORE: " + this.score, 320, 230); ctx.fillText("PRESS 'R' TO RESTART LEVEL", 275, 280);
                ctx.fillStyle = "#00ffcc"; ctx.fillText("PRESS 'M' TO MAIN MENU", 295, 320);
            }
        }
    }

    window.addEventListener("load", () => {
        const game = new Game("game");
        game.scene.change(new MenuScene(game));
        function loop() { game.update(); game.render(); requestAnimationFrame(loop); }
        requestAnimationFrame(loop);
    });
    </script>
</body>
</html>
