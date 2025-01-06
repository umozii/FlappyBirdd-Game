const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

// Set up display
const WIDTH = 500; // Game canvas width  遊戲畫布寬度
const HEIGHT = 600; // Game canvas height  遊戲畫布高度
canvas.width = WIDTH;
canvas.height = HEIGHT;

// Define colors
const WHITE = "#FFFFFF";
const GREEN = "#00FF00";
const BLACK = "#000000";

// Define constants  定義常數
const FPS = 60; // Frames per second 每秒影格數
const GRAVITY = 0.65; // Gravity effect 重力影響
const JUMP = -12; // Jump velocity 跳躍速度
const PIPE_WIDTH = 70; // Pipe width  水管寬度
const MIN_PIPE_GAP = 230; // Minimum vertical gap between pipes  水管間最小垂直間隙
const MAX_PIPE_GAP = 300; // Maximum vertical gap between pipes  水管間最大垂直間隙
const MAX_FALL_SPEED = 4; // Maximum fall speed   最大下墜速度
const PIPE_SPAWN_INTERVAL = 95; // Frames between pipe spawns  產生新水管的幀數間隔

// Game state
const STATES = {
    START: "START",
    COUNTDOWN: "COUNTDOWN",
    PLAYING: "PLAYING",
    GAME_OVER: "GAME_OVER",
};

let state = STATES.START;

// Bird class
class Bird {
    constructor() {
        this.x = WIDTH * 0.15; // 小鳥水平位置
        this.y = HEIGHT / 2;   // 小鳥垂直位置
        this.vel = 0;          // 初始垂直速度
        this.width = 30;       // 小鳥的寬度
        this.height = 30;      // 小鳥的高度
    }

    update() {
        this.vel += GRAVITY; // 受到重力影響
        if (this.vel > MAX_FALL_SPEED) {
            this.vel = MAX_FALL_SPEED; // 限制最大墜落速度
        }
        this.y += this.vel; // 更新小鳥的垂直位置
        if (this.y < 0) {   // 防止小鳥飛出畫面頂部
            this.y = 0;
            this.vel = 0;
        }
    }

    jump() {
        this.vel = JUMP; // 跳躍時重置垂直速度
    }

    draw() {
        ctx.fillStyle = "#FFFF00"; // 小鳥顏色
        ctx.fillRect(this.x, this.y, this.width, this.height);
    }
}

// Pipe class
class Pipe {
    constructor() {
        this.x = WIDTH;
        this.gap = Math.floor(Math.random() * (MAX_PIPE_GAP - MIN_PIPE_GAP + 1)) + MIN_PIPE_GAP;
        this.height = Math.floor(Math.random() * (HEIGHT - this.gap - 100)) + 100;
        this.speed = 2.5; // 調整此值控制速度，例如設為 3 表示較慢的移動速度
    }

    update() {
        this.x -= this.speed;
    }

    draw() {
        ctx.fillStyle = GREEN;
        ctx.fillRect(this.x, 0, PIPE_WIDTH, this.height);
        ctx.fillRect(this.x, this.height + this.gap, PIPE_WIDTH, HEIGHT - this.height - this.gap);
    }
}

// Initialize game objects
const bird = new Bird();
const pipes = [];
let score = 0;
let highScore = 0;
let frameCount = 0;

// Utility functions
function resetGame() {
    bird.y = HEIGHT / 2;
    bird.vel = 0;
    pipes.length = 0;
    score = 0;
    frameCount = 0;
    state = STATES.START;
}

// === Start screen ===
function startScreen() {
    // Fill the entire canvas with white
    ctx.fillStyle = WHITE;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);
  
    // Center alignment for text
    ctx.textAlign = "center";
    ctx.fillStyle = BLACK;
  
    // FlapCube! (48px)
    ctx.font = "48px Arial";
    ctx.fillText("FlapCube!", WIDTH / 2, HEIGHT / 3);
  
    // Press Enter to Start (24px)
    ctx.font = "24px Arial";
    ctx.fillText("Press Enter or Tap to Start", WIDTH / 2, HEIGHT / 2);
  
    // Press up arrow to control (16px)
    ctx.font = "16px Arial";
    ctx.fillText("Press Up Arrow or Tap to Jump", WIDTH / 2, HEIGHT / 2 + 40);
  
    // Developed by umozii (10px) at bottom-right
    ctx.font = "10px Arial";
    ctx.textAlign = "right";
    ctx.fillText("Developed by umozii", WIDTH - 10, HEIGHT - 10);
  }

// function startScreen() {
//     ctx.fillStyle = WHITE;
//     ctx.fillRect(0, 0, WIDTH, HEIGHT);

//     ctx.fillStyle = BLACK;
//     ctx.font = "48px Arial";
//     ctx.fillText("FlapCube!", WIDTH / 2 - 120, HEIGHT / 3);

//     ctx.font = "24px Arial";
//     ctx.fillText("Press Enter to Start", WIDTH / 2 - 100, HEIGHT / 2);
// }

function countdownScreen(count) {
    ctx.fillStyle = WHITE;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);

    ctx.fillStyle = BLACK;
    ctx.font = "48px Arial";
    ctx.textBaseline = "middle";
	
    const text = count.toString();
    const metrics = ctx.measureText(text);

    if (
    typeof metrics.actualBoundingBoxLeft === "number" &&
    typeof metrics.actualBoundingBoxRight === "number"
  ) {
    // 整個文字內容的實際寬度 (不包含字形外的空白)
    const fullWidth = metrics.actualBoundingBoxLeft + metrics.actualBoundingBoxRight;
    // 算出文字「視覺上」的正中點應該在哪
    const x = (WIDTH / 2 - metrics.actualBoundingBoxLeft) + 30;
    const y = HEIGHT / 2;

    ctx.fillText(text, x, y);

  } else {
    // fallback：舊作法，用整段字串的總寬度
    const textWidth = metrics.width;
    const x = (WIDTH - textWidth) / 2;
    const y = HEIGHT / 2;

    ctx.fillText(text, x, y);
  }
}
    
function gameOverScreen() {
    ctx.fillStyle = WHITE;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);

    ctx.fillStyle = BLACK;
    ctx.font = "48px Arial";
    ctx.fillText("Game Over", WIDTH / 2 - 120, HEIGHT / 3);

    ctx.font = "24px Arial";
    ctx.fillText(`Score: ${score}`, WIDTH / 2 - 50, HEIGHT / 2 - 50);
    ctx.fillText(`High Score: ${highScore}`, WIDTH / 2 - 80, HEIGHT / 2);
	
    ctx.font = "24px Arial";
    ctx.fillText("Press Enter to Restart", WIDTH / 2 - 120, HEIGHT / 2 + 50);
}

// Main game loop
function gameLoop() {
    ctx.fillStyle = WHITE;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);

    if (state === STATES.START) {
        startScreen();
    } else if (state === STATES.COUNTDOWN) {
        let countdown = 3 - Math.floor(frameCount / FPS);
        if (countdown <= 0) {
            state = STATES.PLAYING;
        } else {
            countdownScreen(countdown);
        }
    } else if (state === STATES.PLAYING) {
        // Update bird
        bird.update();

        // Spawn pipes
        if (frameCount % PIPE_SPAWN_INTERVAL === 0) {
            pipes.push(new Pipe());
        }

        // Update pipes
        for (let i = pipes.length - 1; i >= 0; i--) {
            pipes[i].update();
            if (pipes[i].x + PIPE_WIDTH < 0) {
                pipes.splice(i, 1);
                score++;
                if (score > highScore) highScore = score;
            }
        }

        // Collision detection
        let collision = false;
        pipes.forEach((pipe) => {
            if (
                bird.x < pipe.x + PIPE_WIDTH &&
                bird.x + bird.width > pipe.x &&
                (bird.y < pipe.height || bird.y + bird.height > pipe.height + pipe.gap)
            ) {
                collision = true;
            }
        });

        if (collision || bird.y > HEIGHT || bird.y < 0) {
            state = STATES.GAME_OVER;
        }

        // Draw everything
        bird.draw();
        pipes.forEach((pipe) => pipe.draw());

        // Draw score
        // 繪製分數與高分
	ctx.fillStyle = BLACK;
	ctx.font = "24px Arial";

	// 繪製左上角的 Score
	ctx.textAlign = "left";
	ctx.fillText(`Score: ${score}`, 10, 30);

	// 計算 High Score 的寬度並動態調整位置
	const highScoreText = `High Score: ${highScore}`;
	const highScoreWidth = ctx.measureText(highScoreText).width;
	ctx.fillText(highScoreText, WIDTH - highScoreWidth - 10, 30); // 保留 10px 邊距

    } else if (state === STATES.GAME_OVER) {
        gameOverScreen();
    }

    frameCount++;
    requestAnimationFrame(gameLoop);
}

// Event listeners
document.addEventListener("keydown", (e) => {
    if (state === STATES.START && e.code === "Enter") {
        state = STATES.COUNTDOWN;
        frameCount = 0;
    } else if (state === STATES.PLAYING && e.code === "ArrowUp") {
        bird.jump();
    } else if (state === STATES.GAME_OVER && e.code === "Enter") {
        resetGame();
    }
});
// canvas.addEventListener("click", () => {
//     if (state === STATES.START) {
//         state = STATES.COUNTDOWN;
//         frameCount = 0;
//     } else if (state === STATES.PLAYING) {
//         bird.jump();
//     } else if (state === STATES.GAME_OVER) {
//         resetGame();
//     }
// });

// canvas.addEventListener("touchstart", (e) => {
//     e.preventDefault(); // 防止手機瀏覽器產生選字/捲動等預設行為
//     if (state === STATES.START) {
//         state = STATES.COUNTDOWN;
//         frameCount = 0;
//     } else if (state === STATES.PLAYING) {
//         bird.jump();
//     } else if (state === STATES.GAME_OVER) {
//         resetGame();
//     }
// });
canvas.addEventListener("pointerdown", (e) => {
    // optional:  e.preventDefault(); // 防止手機或平板的預設行為(捲動/縮放)
    // 確認pointerType
    console.log("Pointer type:", e.pointerType);
    // 可能是 "mouse", "touch", "pen"
    if (state === STATES.START) {
        state = STATES.COUNTDOWN;
        frameCount = 0;
    } else if (state === STATES.PLAYING) {
        bird.jump();
    } else if (state === STATES.GAME_OVER) {
        resetGame();
    }
});


// Start the game loop
gameLoop();