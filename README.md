# HandTracker
<!DOCTYPE html>
<html>
<head>
    <title>Magic Hand Tracking</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; }
        #webcam, #output_canvas {
            position: absolute;
            left: 0; top: 0;
            width: 100vw; height: 100vh;
            object-fit: cover;
            transform: scaleX(-1);
        }
        #output_canvas { z-index: 10; }
    </style>
</head>
<body>
    <video id="webcam" autoplay playsinline></video>
    <canvas id="output_canvas"></canvas>

<script type="module">
    import { HandLandmarker, FilesetResolver, DrawingUtils } from "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3";

    const video = document.getElementById("webcam");
    const canvas = document.getElementById("output_canvas");
    const ctx = canvas.getContext("2d");
    let handLandmarker;
    let magicCircles = []; // Magic circles store karne ke liye

    class MagicCircle {
        constructor(x, y) {
            this.x = x;
            this.y = y;
            this.radius = 0;
            this.maxRadius = 150;
            this.opacity = 1;
            this.rotation = 0;
            this.sparks = [];
        }

        update() {
            this.radius += 8;
            this.rotation += 0.1;
            this.opacity -= 0.015;

            // Sparks generate karo
            if (this.radius < this.maxRadius) {
                for (let i = 0; i < 3; i++) {
                    this.sparks.push({
                        angle: Math.random() * Math.PI * 2,
                        dist: this.radius,
                        life: 1
                    });
                }
            }

            // Sparks update
            this.sparks.forEach(s => s.life -= 0.05);
            this.sparks = this.sparks.filter(s => s.life > 0);
        }

        draw(ctx) {
            ctx.save();
            ctx.translate(this.x, this.y);
            ctx.rotate(this.rotation);

            // Main glow circle
            ctx.beginPath();
            ctx.arc(0, 0, this.radius, 0, Math.PI * 2);
            ctx.strokeStyle = `rgba(255, 165, 0, ${this.opacity})`;
            ctx.lineWidth = 4;
            ctx.shadowColor = "#FFA500";
            ctx.shadowBlur = 30;
            ctx.stroke();

            // Inner rotating lines
            for (let i = 0; i < 8; i++) {
                ctx.beginPath();
                ctx.moveTo(0, 0);
                ctx.lineTo(this.radius * 0.8, 0);
                ctx.strokeStyle = `rgba(255, 215, 0, ${this.opacity * 0.6})`;
                ctx.lineWidth = 2;
                ctx.rotate(Math.PI / 4);
                ctx.stroke();
            }

            // Sparks
            this.sparks.forEach(s => {
                const sx = Math.cos(s.angle) * s.dist;
                const sy = Math.sin(s.angle) * s.dist;
                ctx.beginPath();
                ctx.arc(sx, sy, 3, 0, Math.PI * 2);
                ctx.fillStyle = `rgba(255, 255, 0, ${s.life})`;
                ctx.fill();
            });

            ctx.restore();
        }

        isDead() {
            return this.opacity <= 0;
        }
    }

    async function init() {
        const vision = await FilesetResolver.forVisionTasks(
            "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3/wasm"
        );
        
        handLandmarker = await HandLandmarker.createFromOptions(vision, {
            baseOptions: {
                modelAssetPath: "https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task"
            },
            runningMode: "VIDEO",
            numHands: 2
        });

        const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 1280, height: 720 } });
        video.srcObject = stream;
        video.addEventListener("loadeddata", predictWebcam);
    }

    function getDistance(p1, p2) {
        return Math.sqrt(Math.pow(p1.x - p2.x, 2) + Math.pow(p1.y - p2.y, 2));
    }

    async function predictWebcam() {
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        const drawingUtils = new DrawingUtils(ctx);
        
        const results = handLandmarker.detectForVideo(video, performance.now());
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        if (results.landmarks) {
            for (const landmarks of results.landmarks) {
                // Normal hand skeleton - thoda transparent
                drawingUtils.drawConnectors(landmarks, HandLandmarker.HAND_CONNECTIONS, { 
                    color: "rgba(255, 165, 0, 0.3)", lineWidth: 2 
                });
                drawingUtils.drawLandmarks(landmarks, { 
                    color: "rgba(255, 165, 0, 0.5)", radius: 3 
                });

                // Pinch detect karo: Thumb tip 4 aur Index tip 8
                const thumb = landmarks[4];
                const index = landmarks[8];
                const distance = getDistance(thumb, index);

                // Agar ungliyaan mili hui hain
                if (distance < 0.05) {
                    const midX = (thumb.x + index.x) / 2 * canvas.width;
                    const midY = (thumb.y + index.y) / 2 * canvas.height;
                    magicCircles.push(new MagicCircle(midX, midY));
                }
            }
        }

        // Saare magic circles draw aur update karo
        magicCircles.forEach(circle => {
            circle.update();
            circle.draw(ctx);
        });
        magicCircles = magicCircles.filter(circle =>!circle.isDead());

        requestAnimationFrame(predictWebcam);
    }

    init();
</script> 
</body>
</html>
