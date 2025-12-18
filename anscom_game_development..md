<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>PHASE LAG | Prediction Engine</title>
    <style>
        /* 
         * ==========================================
         * CORE STYLES
         * Minimalist, High Contrast, Typography
         * ==========================================
         */
        :root {
            --bg: #080808;
            --fg: #e0e0e0;
            --accent-ok: #00ffaa;
            --accent-warn: #ffcc00;
            --accent-bad: #ff3333;
            --accent-ghost: #444455;
            --font-main: 'Courier New', Courier, monospace;
        }

        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
            cursor: crosshair;
        }

        body, html {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: var(--bg);
            color: var(--fg);
            font-family: var(--font-main);
            overscroll-behavior: none;
        }

        #game-container {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            justify-content: center;
            align-items: center;
        }

        canvas {
            display: block;
            /* Rendering optimization */
            image-rendering: pixelated; 
        }

        /* UI OVERLAYS */
        .ui-layer {
            position: absolute;
            pointer-events: none;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            padding: 2rem;
            mix-blend-mode: difference;
        }

        .header-stats {
            display: flex;
            justify-content: space-between;
            font-size: 14px;
            letter-spacing: 2px;
            text-transform: uppercase;
            opacity: 0.8;
        }

        .center-msg {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            pointer-events: none;
            transition: opacity 0.3s ease;
        }

        h1 {
            font-size: 4rem;
            font-weight: 100;
            margin: 0;
            letter-spacing: -2px;
            opacity: 0.9;
        }

        p.sub {
            font-size: 0.8rem;
            letter-spacing: 4px;
            margin-top: 1rem;
            color: var(--accent-warn);
            animation: pulse 2s infinite;
        }

        .debug-info {
            font-size: 10px;
            color: #555;
            text-align: right;
        }

        /* ANIMATIONS */
        @keyframes pulse {
            0% { opacity: 0.4; }
            50% { opacity: 1; }
            100% { opacity: 0.4; }
        }

        .glitch {
            animation: glitch-anim 0.3s infinite;
        }

        @keyframes glitch-anim {
            0% { transform: translate(0) }
            20% { transform: translate(-2px, 2px) }
            40% { transform: translate(-2px, -2px) }
            60% { transform: translate(2px, 2px) }
            80% { transform: translate(2px, -2px) }
            100% { transform: translate(0) }
        }

        /* HIDDEN ELEMENTS */
        .hidden {
            opacity: 0;
            pointer-events: none;
        }
        
    </style>
</head>
<body>

    <div id="game-container">
        <canvas id="stage"></canvas>
        <div class="ui-layer">
            <div class="header-stats">
                <div id="score-display">SYNC: 0.00%</div>
                <div id="lag-display">LAG: 0ms</div>
                <div id="combo-display">SEQ: 0</div>
            </div>
            
            <div id="start-screen" class="center-msg">
                <h1>PHASE LAG</h1>
                <p class="sub">CLICK TO SYNC</p>
                <p style="font-size: 10px; margin-top: 30px; color:#666">USE HEADPHONES // PREDICT THE FUTURE</p>
            </div>

            <div class="debug-info">
                <span id="fps-display">FPS: 60</span> | <span id="difficulty-display">DIFF: 1.0</span>
            </div>
        </div>
    </div>

<script>
/**
 * ============================================================================
 * PHASE LAG - ENGINE ARCHITECTURE
 * ============================================================================
 * 
 * 1. UTILS & MATH
 * 2. AUDIO ENGINE (Procedural Synthesis)
 * 3. INPUT SYSTEM
 * 4. TIME & LOOP
 * 5. PHYSICS STATE (The "Real" World)
 * 6. LAG BUFFER (The "Past" World)
 * 7. MOTION GENERATORS
 * 8. GAME CONTROLLER (Rules, Scoring, Difficulty)
 * 9. RENDERER (Canvas)
 * 
 * Designed for 120Hz operation where available.
 * Zero-allocation in hot loops enforced manually.
 */

// ============================================================================
// 1. UTILS & MATH
// ============================================================================

const M = {
    PI: Math.PI,
    TWO_PI: Math.PI * 2,
    
    clamp: (v, min, max) => Math.max(min, Math.min(max, v)),
    
    lerp: (a, b, t) => a + (b - a) * t,
    
    inverseLerp: (a, b, v) => (v - a) / (b - a),
    
    map: (v, iMin, iMax, oMin, oMax) => 
        oMin + (oMax - oMin) * ((v - iMin) / (iMax - iMin)),
    
    randomRange: (min, max) => Math.random() * (max - min) + min,

    // Smoothstep for softer transitions
    smoothstep: (min, max, value) => {
        const x = Math.max(0, Math.min(1, (value - min) / (max - min)));
        return x * x * (3 - 2 * x);
    },

    // 1D Perlin-like Noise (Simplified for performance)
    noise: (function() {
        const MAX_VERTICES = 256;
        const MAX_VERTICES_MASK = MAX_VERTICES -1;
        const amplitude = 1;
        const scale = 1;
        const r = [];
        for ( let i = 0; i < MAX_VERTICES; ++i ) {
            r.push(Math.random());
        }
        return function(x) {
            const scaledX = x * scale;
            const xFloor = Math.floor(scaledX);
            const t = scaledX - xFloor;
            const tRemapSmoothstep = t * t * ( 3 - 2 * t );
            const xMin = xFloor & MAX_VERTICES_MASK;
            const xMax = ( xMin + 1 ) & MAX_VERTICES_MASK;
            const y = M.lerp( r[ xMin ], r[ xMax ], tRemapSmoothstep );
            return y * amplitude;
        };
    })()
};

// ============================================================================
// 2. AUDIO ENGINE (WEB AUDIO API)
// ============================================================================

class AudioEngine {
    constructor() {
        this.ctx = null;
        this.masterGain = null;
        this.isInit = false;
        
        // Procedural generation parameters
        this.baseFreq = 55.0; // A1
    }

    init() {
        if (this.isInit) return;
        
        const AudioContext = window.AudioContext || window.webkitAudioContext;
        this.ctx = new AudioContext();
        
        this.masterGain = this.ctx.createGain();
        this.masterGain.gain.value = 0.4;
        this.masterGain.connect(this.ctx.destination);
        
        // Compressor to glue sounds and prevent clipping
        this.compressor = this.ctx.createDynamicsCompressor();
        this.compressor.threshold.value = -10;
        this.compressor.knee.value = 30;
        this.compressor.ratio.value = 12;
        this.compressor.attack.value = 0.003;
        this.compressor.release.value = 0.25;
        this.compressor.connect(this.masterGain);

        this.startAmbience();
        this.isInit = true;
    }

    startAmbience() {
        // Low Drone
        const osc = this.ctx.createOscillator();
        osc.type = 'sawtooth';
        osc.frequency.value = this.baseFreq;
        
        const filter = this.ctx.createBiquadFilter();
        filter.type = 'lowpass';
        filter.frequency.value = 120;
        filter.Q.value = 1;

        const lfo = this.ctx.createOscillator();
        lfo.type = 'sine';
        lfo.frequency.value = 0.1; // Slow undulation
        
        const lfoGain = this.ctx.createGain();
        lfoGain.gain.value = 50; // Filter modulation depth

        lfo.connect(lfoGain);
        lfoGain.connect(filter.frequency);
        
        osc.connect(filter);
        filter.connect(this.compressor);
        
        osc.start();
        lfo.start();

        // High Ethereal Shimmer
        const osc2 = this.ctx.createOscillator();
        osc2.type = 'sine';
        osc2.frequency.value = this.baseFreq * 4;
        const gain2 = this.ctx.createGain();
        gain2.gain.value = 0.05;
        osc2.connect(gain2);
        gain2.connect(this.compressor);
        osc2.start();
    }

    playClick() {
        if (!this.ctx) return;
        const t = this.ctx.currentTime;
        const osc = this.ctx.createOscillator();
        const gain = this.ctx.createGain();
        
        osc.type = 'triangle';
        osc.frequency.setValueAtTime(800, t);
        osc.frequency.exponentialRampToValueAtTime(1200, t + 0.05);
        
        gain.gain.setValueAtTime(0, t);
        gain.gain.linearRampToValueAtTime(0.3, t + 0.01);
        gain.gain.exponentialRampToValueAtTime(0.001, t + 0.1);
        
        osc.connect(gain);
        gain.connect(this.compressor);
        
        osc.start(t);
        osc.stop(t + 0.1);
    }

    playSuccess(combo) {
        if (!this.ctx) return;
        const t = this.ctx.currentTime;
        
        // Harmonic scaling based on combo
        const notes = [1, 1.2, 1.25, 1.5, 1.66, 2.0]; // Major pentatonic ratios
        const ratio = notes[combo % notes.length] * (1 + Math.floor(combo / notes.length));
        const freq = 440 * ratio;

        const osc = this.ctx.createOscillator();
        const gain = this.ctx.createGain();
        
        osc.type = 'sine';
        osc.frequency.setValueAtTime(freq, t);
        
        gain.gain.setValueAtTime(0, t);
        gain.gain.linearRampToValueAtTime(0.4, t + 0.02);
        gain.gain.exponentialRampToValueAtTime(0.001, t + 0.4);
        
        osc.connect(gain);
        gain.connect(this.compressor);
        
        osc.start(t);
        osc.stop(t + 0.5);

        // Sub bass thump
        const kick = this.ctx.createOscillator();
        const kGain = this.ctx.createGain();
        kick.frequency.setValueAtTime(150, t);
        kick.frequency.exponentialRampToValueAtTime(0.01, t + 0.2);
        kGain.gain.setValueAtTime(0.5, t);
        kGain.gain.exponentialRampToValueAtTime(0.001, t + 0.2);
        kick.connect(kGain);
        kGain.connect(this.compressor);
        kick.start(t);
        kick.stop(t + 0.2);
    }

    playFail() {
        if (!this.ctx) return;
        const t = this.ctx.currentTime;
        
        const osc = this.ctx.createOscillator();
        const gain = this.ctx.createGain();
        
        osc.type = 'sawtooth';
        osc.frequency.setValueAtTime(100, t);
        osc.frequency.linearRampToValueAtTime(50, t + 0.3);
        
        // Distortion effect
        const shaper = this.ctx.createWaveShaper();
        shaper.curve = this.makeDistortionCurve(400);

        gain.gain.setValueAtTime(0.3, t);
        gain.gain.exponentialRampToValueAtTime(0.001, t + 0.3);
        
        osc.connect(shaper);
        shaper.connect(gain);
        gain.connect(this.compressor);
        
        osc.start(t);
        osc.stop(t + 0.4);
    }

    // Distortion curve helper
    makeDistortionCurve(amount) {
        const k = typeof amount === 'number' ? amount : 50,
            n_samples = 44100,
            curve = new Float32Array(n_samples),
            deg = Math.PI / 180;
        let i = 0, x;
        for ( ; i < n_samples; ++i ) {
            x = i * 2 / n_samples - 1;
            curve[i] = ( 3 + k ) * x * 20 * deg / ( Math.PI + k * Math.abs(x) );
        }
        return curve;
    }
}

// ============================================================================
// 3. INPUT SYSTEM
// ============================================================================

class InputSystem {
    constructor() {
        this.active = false;
        this.triggered = false;
        this.x = 0;
        this.y = 0;
        
        this.binds();
    }

    binds() {
        const trigger = (e) => {
            if (e.type === 'keydown' && e.code !== 'Space') return;
            // Prevent default spacebar scrolling
            if (e.type === 'keydown') e.preventDefault();
            this.triggered = true;
        };

        window.addEventListener('keydown', trigger);
        window.addEventListener('mousedown', trigger);
        window.addEventListener('touchstart', (e) => {
            e.preventDefault(); // Stop zoom/scroll
            trigger(e);
        }, {passive: false});

        window.addEventListener('mousemove', (e) => {
            this.x = e.clientX;
            this.y = e.clientY;
        });
    }

    // Consumes the input (returns true once per press)
    consume() {
        if (this.triggered) {
            this.triggered = false;
            return true;
        }
        return false;
    }
}

// ============================================================================
// 4. PHYSICS & MOTION ENGINE
// ============================================================================

const MOTION_TYPES = {
    LINEAR_PINGPONG: 0,
    SINE_WAVE: 1,
    ACCELERATED_BOUNCE: 2,
    NOISE_WANDER: 3,
    LISSAJOUS_1D: 4,
    CHAOTIC: 5
};

class MotionEngine {
    constructor() {
        this.reset();
    }

    reset() {
        this.t = 0;
        this.pos = 0.5; // Normalized 0.0 to 1.0
        this.vel = 0.01;
        this.type = MOTION_TYPES.LINEAR_PINGPONG;
        this.params = {
            speed: 1.0,
            amplitude: 0.4,
            center: 0.5,
            frequency: 1.0,
            phase: 0
        };
        // For chaotic attractors
        this.chaosState = { x: 0.1, y: 0, z: 0 }; 
    }

    setType(type, difficulty) {
        this.type = type;
        this.t = 0;
        // Scale parameters by difficulty
        const speedMod = 1 + (difficulty * 0.2);
        
        switch(type) {
            case MOTION_TYPES.LINEAR_PINGPONG:
                this.params.speed = 0.015 * speedMod;
                break;
            case MOTION_TYPES.SINE_WAVE:
                this.params.frequency = 2.0 * speedMod;
                this.params.amplitude = 0.4;
                break;
            case MOTION_TYPES.NOISE_WANDER:
                this.params.speed = 0.01 * speedMod;
                break;
            case MOTION_TYPES.ACCELERATED_BOUNCE:
                this.vel = 0.02 * speedMod;
                this.pos = 0.5;
                break;
            case MOTION_TYPES.LISSAJOUS_1D:
                this.params.frequency = 1.5 * speedMod;
                break;
            case MOTION_TYPES.CHAOTIC:
                this.chaosState = {x: 0.1, y: 0.1, z: 0.1};
                this.params.speed = 0.01 * speedMod;
                break;
        }
    }

    update(dt) {
        this.t += dt; // dt is in seconds

        switch(this.type) {
            case MOTION_TYPES.LINEAR_PINGPONG:
                // Simple bouncing back and forth
                this.pos += this.vel * (dt * 60); 
                if (this.pos > 0.9 || this.pos < 0.1) {
                    this.vel *= -1;
                    this.pos = M.clamp(this.pos, 0.1, 0.9);
                }
                break;

            case MOTION_TYPES.SINE_WAVE:
                // Sinusoidal
                this.pos = this.params.center + Math.sin(this.t * this.params.frequency) * this.params.amplitude;
                break;

            case MOTION_TYPES.ACCELERATED_BOUNCE:
                // Gravity-like bounce
                const gravity = 0.002 * (dt * 60);
                this.vel += (this.pos < 0.5 ? gravity : -gravity); // Pull to center? No, let's pull to edges
                // Actually, let's do a spring to center
                const force = (0.5 - this.pos) * 0.005;
                this.vel += force * (dt * 60);
                // Drag
                this.vel *= 0.99;
                
                // Add velocity
                this.pos += this.vel * (dt * 60) * 10;
                
                // Hard bounds bounce
                if (this.pos < 0.1) { this.pos = 0.1; this.vel *= -1; }
                if (this.pos > 0.9) { this.pos = 0.9; this.vel *= -1; }
                break;

            case MOTION_TYPES.NOISE_WANDER:
                // Perlin noise based
                const n = M.noise(this.t * this.params.frequency); // 0..1
                this.pos = 0.2 + (n * 0.6);
                break;
            
            case MOTION_TYPES.LISSAJOUS_1D:
                // Combined Sines
                const s1 = Math.sin(this.t * this.params.frequency);
                const s2 = Math.cos(this.t * this.params.frequency * 1.5 + 0.4);
                this.pos = 0.5 + (s1 + s2) * 0.22;
                break;

            case MOTION_TYPES.CHAOTIC:
                // Simplified Lorenz-like projection or Logistic Map
                // Let's use Logistic Map logic but smoothed over time
                // x(n+1) = r * x(n) * (1 - x(n))
                // We simulate this continuously
                
                // Using Duffing oscillator approximation for 1D movement
                // dx = v
                // dv = x - x^3 - delta*v + gamma*cos(omega*t)
                const gamma = 0.3;
                const omega = 1.2;
                const delta = 0.2;
                const dxdt = this.chaosState.y;
                const dydt = this.chaosState.x - Math.pow(this.chaosState.x, 3) - delta * this.chaosState.y + gamma * Math.cos(omega * this.t * 5);
                
                this.chaosState.x += dxdt * dt * 5;
                this.chaosState.y += dydt * dt * 5;
                
                // Map x (-1.5 to 1.5) to 0..1
                this.pos = M.map(this.chaosState.x, -1.5, 1.5, 0.1, 0.9);
                this.pos = M.clamp(this.pos, 0.05, 0.95);
                break;
        }

        return this.pos;
    }
}

// ============================================================================
// 5. LAG SYSTEM (CIRCULAR BUFFER)
// ============================================================================

class LagSystem {
    constructor() {
        // Store 5 seconds of history at 120hz approx
        this.capacity = 600; 
        this.buffer = new Float32Array(this.capacity);
        this.times = new Float64Array(this.capacity);
        this.head = 0;
        this.length = 0;
    }

    push(val, time) {
        this.buffer[this.head] = val;
        this.times[this.head] = time;
        this.head = (this.head + 1) % this.capacity;
        if (this.length < this.capacity) this.length++;
    }

    // Get value at (currentTime - delaySeconds)
    getDelayedValue(currentTime, delay) {
        if (this.length === 0) return 0.5;

        const targetTime = currentTime - delay;
        
        // Linear scan backwards from head is efficient enough for small buffers
        // and short delays. Binary search is overkill for < 600 items usually.
        
        let idx = (this.head - 1 + this.capacity) % this.capacity;
        
        // If target time is newer than most recent, return most recent
        if (targetTime >= this.times[idx]) return this.buffer[idx];

        for (let i = 0; i < this.length; i++) {
            const currIdx = (this.head - 1 - i + this.capacity) % this.capacity;
            const prevIdx = (this.head - 1 - i - 1 + this.capacity) % this.capacity;
            
            const t2 = this.times[currIdx];
            const t1 = this.times[prevIdx];
            
            if (targetTime >= t1 && targetTime <= t2) {
                // Found bracket
                const factor = (targetTime - t1) / (t2 - t1);
                return M.lerp(this.buffer[prevIdx], this.buffer[currIdx], factor);
            }
        }

        // Too old, return oldest
        const tail = (this.head - this.length + this.capacity) % this.capacity;
        return this.buffer[tail];
    }
}

// ============================================================================
// 6. RENDERER
// ============================================================================

class Renderer {
    constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d', { alpha: false });
        this.width = 0;
        this.height = 0;
        
        this.resize();
        window.addEventListener('resize', () => this.resize());
        
        this.particles = [];
    }

    resize() {
        this.width = window.innerWidth;
        this.height = window.innerHeight;
        this.canvas.width = this.width;
        this.canvas.height = this.height;
    }

    clear() {
        this.ctx.fillStyle = '#080808';
        this.ctx.fillRect(0, 0, this.width, this.height);
    }

    spawnParticle(x, y, color, type) {
        for(let i=0; i<8; i++) {
            this.particles.push({
                x: x,
                y: y,
                vx: M.randomRange(-2, 2),
                vy: M.randomRange(-2, 2),
                life: 1.0,
                color: color,
                size: M.randomRange(2, 5)
            });
        }
    }

    drawParticles(dt) {
        for (let i = this.particles.length - 1; i >= 0; i--) {
            const p = this.particles[i];
            p.x += p.vx;
            p.y += p.vy;
            p.life -= dt * 2;
            
            if (p.life <= 0) {
                this.particles.splice(i, 1);
                continue;
            }

            this.ctx.globalAlpha = p.life;
            this.ctx.fillStyle = p.color;
            this.ctx.beginPath();
            this.ctx.arc(p.x, p.y, p.size * p.life, 0, M.TWO_PI);
            this.ctx.fill();
        }
        this.ctx.globalAlpha = 1.0;
    }

    // Main Draw Routine
    render(gameState) {
        this.clear();

        // 1. Draw Static Target Zone
        // The target is a vertical band in the center (or moving, but let's keep it center for now)
        const targetX = this.width * gameState.targetPos;
        const targetWidth = this.width * gameState.targetWidth;
        
        // Target Zone Glow
        const gradient = this.ctx.createLinearGradient(targetX - targetWidth, 0, targetX + targetWidth, 0);
        gradient.addColorStop(0, 'rgba(0, 255, 170, 0)');
        gradient.addColorStop(0.5, 'rgba(0, 255, 170, 0.1)');
        gradient.addColorStop(1, 'rgba(0, 255, 170, 0)');
        
        this.ctx.fillStyle = gradient;
        this.ctx.fillRect(targetX - targetWidth * 2, 0, targetWidth * 4, this.height);

        // Target Lines
        this.ctx.strokeStyle = '#00ffaa';
        this.ctx.lineWidth = 2;
        this.ctx.beginPath();
        this.ctx.moveTo(targetX, 0);
        this.ctx.lineTo(targetX, this.height);
        this.ctx.stroke();

        this.ctx.lineWidth = 1;
        this.ctx.strokeStyle = 'rgba(0, 255, 170, 0.3)';
        this.ctx.strokeRect(targetX - targetWidth/2, 0, targetWidth, this.height);

        // 2. Draw Delayed State (The Ghost/Player View)
        // This is what the player sees
        const delayedX = this.width * gameState.delayedPos;
        
        // Trail effect for delayed object
        this.ctx.shadowBlur = 15;
        this.ctx.shadowColor = '#e0e0e0';
        this.ctx.fillStyle = '#e0e0e0';
        
        // Shape: Vertical pill
        const cursorH = 60;
        const cursorW = 10;
        
        this.ctx.fillRect(delayedX - cursorW/2, (this.height/2) - cursorH/2, cursorW, cursorH);
        this.ctx.shadowBlur = 0;

        // 3. Visualizing the Lag (Subtle connection line)
        // Only visible when lag is high to indicate "stretching"
        if (gameState.lag > 0.3) {
            this.ctx.strokeStyle = '#ff3333';
            this.ctx.setLineDash([5, 15]);
            this.ctx.globalAlpha = 0.3;
            this.ctx.beginPath();
            // Just a visual cue at the bottom
            this.ctx.moveTo(delayedX, this.height - 50);
            this.ctx.lineTo(targetX, this.height - 50);
            this.ctx.stroke();
            this.ctx.setLineDash([]);
            this.ctx.globalAlpha = 1.0;
        }

        // 4. Draw Feedback (Where the REAL object was on click)
        if (gameState.feedbackTimer > 0) {
            const alpha = gameState.feedbackTimer;
            const realX = this.width * gameState.lastRealPos;
            
            // Draw a ghost of the real position
            this.ctx.strokeStyle = gameState.lastResult === 'HIT' ? '#00ffaa' : '#ff3333';
            this.ctx.lineWidth = 2;
            this.ctx.globalAlpha = alpha;
            this.ctx.strokeRect(realX - 10, (this.height/2) - 40, 20, 80);
            
            // Connection line between where you clicked (delayedX is moving, so we use snapshot)
            // Actually, let's just show the error distance text
            this.ctx.fillStyle = gameState.lastResult === 'HIT' ? '#00ffaa' : '#ff3333';
            this.ctx.font = '12px Courier New';
            this.ctx.textAlign = 'center';
            this.ctx.fillText(gameState.lastResult, realX, (this.height/2) - 50);
            
            this.ctx.globalAlpha = 1.0;
        }

        // 5. Particles
        this.drawParticles(0.016); // Approx dt
        
        // 6. Screen Shake
        if (gameState.shake > 0) {
            const dx = M.randomRange(-gameState.shake, gameState.shake);
            const dy = M.randomRange(-gameState.shake, gameState.shake);
            this.ctx.canvas.style.transform = `translate(${dx}px, ${dy}px)`;
        } else {
            this.ctx.canvas.style.transform = `none`;
        }
    }
}

// ============================================================================
// 7. GAME CONTROLLER
// ============================================================================

class Game {
    constructor() {
        this.audio = new AudioEngine();
        this.input = new InputSystem();
        this.renderer = new Renderer('stage');
        
        this.motion = new MotionEngine();
        this.lagSystem = new LagSystem();
        
        this.state = 'MENU'; // MENU, PLAYING, GAMEOVER
        
        this.score = 0;
        this.combo = 0;
        this.difficulty = 1.0;
        this.lagDuration = 0.2; // Seconds
        
        this.targetPos = 0.5;
        this.targetWidth = 0.1; // 10% of screen width tolerance total
        
        this.realPos = 0.5;
        this.delayedPos = 0.5;
        
        // Feedback visuals
        this.feedbackTimer = 0;
        this.lastRealPos = 0;
        this.lastResult = '';
        this.shake = 0;

        // DOM Elements
        this.uiScore = document.getElementById('score-display');
        this.uiLag = document.getElementById('lag-display');
        this.uiCombo = document.getElementById('combo-display');
        this.uiDiff = document.getElementById('difficulty-display');
        this.uiStart = document.getElementById('start-screen');
        
        // Loop vars
        this.lastTime = 0;
        this.accumulator = 0;
        this.timeStep = 1/120; // High precision physics
    }

    start() {
        this.audio.init();
        this.uiStart.classList.add('hidden');
        this.state = 'PLAYING';
        this.score = 0;
        this.combo = 0;
        this.difficulty = 1.0;
        this.lagDuration = 0.2;
        this.motion.reset();
        
        this.updateUI();
    }

    loop(timestamp) {
        if (!this.lastTime) this.lastTime = timestamp;
        const frameTime = (timestamp - this.lastTime) / 1000;
        this.lastTime = timestamp;

        // Clamp max frame time to avoid spiral of death
        const safeFrameTime = Math.min(frameTime, 0.1);

        this.accumulator += safeFrameTime;

        while (this.accumulator >= this.timeStep) {
            this.updateFixed(this.timeStep, timestamp / 1000);
            this.accumulator -= this.timeStep;
        }

        // Interpolation for rendering could go here, but simple latest state is okay for this style
        this.draw();
        
        requestAnimationFrame((t) => this.loop(t));
    }

    updateFixed(dt, time) {
        if (this.state === 'MENU') {
            // Idle animation in background
            this.realPos = this.motion.update(dt);
            this.lagSystem.push(this.realPos, time);
            this.delayedPos = this.lagSystem.getDelayedValue(time, 0.1);
            
            if (this.input.consume()) {
                this.start();
            }
            return;
        }

        if (this.state === 'PLAYING') {
            // 1. Difficulty Scaling
            // Increase lag over time
            // Increase motion speed over time
            this.difficulty += dt * 0.05; // Linearly increase difficulty
            
            // Lag oscillates to confuse prediction
            const lagOsc = Math.sin(time * 0.5) * 0.05; 
            const baseLag = 0.2 + (this.difficulty * 0.05);
            this.lagDuration = M.clamp(baseLag + lagOsc, 0.1, 1.5);

            // Change Motion Type based on difficulty milestones
            const diffFloor = Math.floor(this.difficulty);
            if (diffFloor % 5 === 0) this.motion.setType(MOTION_TYPES.CHAOTIC, this.difficulty);
            else if (diffFloor % 4 === 0) this.motion.setType(MOTION_TYPES.LISSAJOUS_1D, this.difficulty);
            else if (diffFloor % 3 === 0) this.motion.setType(MOTION_TYPES.ACCELERATED_BOUNCE, this.difficulty);
            else if (diffFloor % 2 === 0) this.motion.setType(MOTION_TYPES.SINE_WAVE, this.difficulty);
            else this.motion.setType(MOTION_TYPES.LINEAR_PINGPONG, this.difficulty);

            // 2. Physics Step
            this.realPos = this.motion.update(dt);
            this.lagSystem.push(this.realPos, time);

            // 3. Resolve Display State
            this.delayedPos = this.lagSystem.getDelayedValue(time, this.lagDuration);

            // 4. Input Handling
            if (this.input.consume()) {
                this.checkHit();
            }

            // 5. FX Decay
            if (this.feedbackTimer > 0) this.feedbackTimer -= dt * 2;
            if (this.shake > 0) this.shake -= dt * 30;
            if (this.shake < 0) this.shake = 0;
            
            // 6. UI Updates (throttled ideally, but lightweight here)
            this.updateUI();
        }
    }

    checkHit() {
        // Core mechanic: Compare REAL pos to TARGET pos
        // The player clicks based on what they SEE (delayed), but they must predict
        // actually, the prompt says "Player must trigger when the real system will align with a target"
        // and "Visible state is delayed".
        
        // Distance between Real Position and Center (0.5)
        // Normalized distance 0..1
        const dist = Math.abs(this.realPos - this.targetPos);
        const hitWindow = this.targetWidth / 2; // +/- from center

        this.lastRealPos = this.realPos;
        this.feedbackTimer = 1.0;

        if (dist < hitWindow) {
            // HIT
            // Precision bonus
            const precision = 1.0 - (dist / hitWindow); // 1.0 is perfect center
            const scoreAdd = Math.floor(100 * precision * (1 + this.combo * 0.1) * this.difficulty);
            
            this.score += scoreAdd;
            this.combo++;
            this.lastResult = 'HIT';
            
            this.audio.playSuccess(this.combo);
            this.renderer.spawnParticle(this.renderer.width * this.realPos, this.renderer.height/2, '#00ffaa');
            
            // Slight shake for impact
            this.shake = 5;

        } else {
            // MISS
            this.combo = 0;
            this.lastResult = 'MISS';
            this.audio.playFail();
            this.shake = 20;
            this.renderer.spawnParticle(this.renderer.width * this.realPos, this.renderer.height/2, '#ff3333');
            
            // Penalty? Score reduction?
            this.score = Math.max(0, this.score - 50);
        }
    }

    updateUI() {
        this.uiScore.innerText = `SCORE: ${this.score.toString().padStart(6, '0')}`;
        this.uiLag.innerText = `LAG: ${(this.lagDuration * 1000).toFixed(0)}ms`;
        this.uiCombo.innerText = `COMBO: ${this.combo}`;
        this.uiDiff.innerText = `DIFF: ${this.difficulty.toFixed(2)}`;
        
        // Color code lag
        if (this.lagDuration > 0.5) this.uiLag.style.color = '#ff3333';
        else if (this.lagDuration > 0.3) this.uiLag.style.color = '#ffcc00';
        else this.uiLag.style.color = '#e0e0e0';
    }

    draw() {
        const renderState = {
            targetPos: this.targetPos,
            targetWidth: this.targetWidth,
            delayedPos: this.delayedPos,
            realPos: this.realPos, // For debug/ghosts
            lag: this.lagDuration,
            feedbackTimer: this.feedbackTimer,
            lastRealPos: this.lastRealPos,
            lastResult: this.lastResult,
            shake: this.shake
        };
        
        this.renderer.render(renderState);
    }
}

// ============================================================================
// BOOTSTRAP
// ============================================================================

window.onload = () => {
    const game = new Game();
    
    // FPS counter logic
    const fpsElem = document.getElementById('fps-display');
    let lastLoop = performance.now();
    let frameCount = 0;
    
    const fpsLoop = () => {
        const now = performance.now();
        frameCount++;
        if (now - lastLoop >= 1000) {
            fpsElem.innerText = `FPS: ${frameCount}`;
            frameCount = 0;
            lastLoop = now;
        }
        requestAnimationFrame(fpsLoop);
    }
    fpsLoop();

    // Start main loop
    requestAnimationFrame((t) => game.loop(t));
};

</script>
</body>
</html>
