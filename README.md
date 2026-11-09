<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport"
content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">

<title>Open World Survival</title>

<style>
* {
    box-sizing: border-box;
    touch-action: none;
}

body {
    margin: 0;
    overflow: hidden;
    background: #000;
    font-family: Arial, sans-serif;
}

#game {
    width: 100vw;
    height: 100vh;
}

/* ================= HUD ================= */

#hud {
    position: fixed;
    top: 15px;
    left: 15px;
    z-index: 10;
    color: white;
    text-shadow: 2px 2px 3px black;
}

.label {
    font-size: 13px;
    margin-top: 5px;
}

.bar-box {
    width: 190px;
    height: 18px;
    background: rgba(0,0,0,0.65);
    border: 2px solid white;
    border-radius: 10px;
    overflow: hidden;
}

.bar {
    width: 100%;
    height: 100%;
    transition: width 0.3s;
}

#health { background: #e53935; }
#hunger { background: #f5a623; }
#thirst { background: #2196f3; }

#inventory {
    position: fixed;
    top: 15px;
    right: 15px;
    z-index: 10;
    color: white;
    background: rgba(0,0,0,0.55);
    padding: 12px;
    border-radius: 10px;
    text-align: center;
}

#message {
    position: fixed;
    left: 50%;
    top: 20%;
    transform: translateX(-50%);
    color: white;
    font-size: 18px;
    text-shadow: 2px 2px 4px black;
    z-index: 20;
    pointer-events: none;
}

/* ================= MOVEMENT CONTROLS ================= */

#controls {
    position: fixed;
    left: 20px;
    bottom: 20px;
    width: 175px;
    height: 175px;
    z-index: 20;
}

.btn {
    position: absolute;
    width: 60px;
    height: 60px;
    border: 3px solid rgba(255,255,255,0.8);
    background: rgba(30,30,30,0.7);
    color: white;
    font-size: 27px;
    border-radius: 15px;
    display: flex;
    align-items: center;
    justify-content: center;
    user-select: none;
}

#up { left: 58px; top: 0; }
#left { left: 0; top: 58px; }
#right { right: 0; top: 58px; }
#down { left: 58px; bottom: 0; }

/* ACTION BUTTON */

#actionButton {
    position: fixed;
    right: 25px;
    bottom: 35px;
    width: 85px;
    height: 85px;
    border-radius: 50%;
    border: 3px solid white;
    background: rgba(30,100,40,0.8);
    color: white;
    font-size: 15px;
    z-index: 30;
}

#cameraText {
    position: fixed;
    bottom: 20px;
    right: 125px;
    color: white;
    background: rgba(0,0,0,0.45);
    padding: 8px;
    border-radius: 8px;
    font-size: 12px;
}

</style>
</head>

<body>

<div id="game"></div>

<!-- HUD -->
<div id="hud">

    <div class="label">❤️ HEALTH</div>
    <div class="bar-box">
        <div class="bar" id="health"></div>
    </div>

    <div class="label">🍗 HUNGER</div>
    <div class="bar-box">
        <div class="bar" id="hunger"></div>
    </div>

    <div class="label">💧 THIRST</div>
    <div class="bar-box">
        <div class="bar" id="thirst"></div>
    </div>

</div>

<div id="inventory">
    🎒 INVENTORY<br>
    🍎 Food: <span id="foodCount">0</span>
</div>

<div id="message"></div>

<!-- MOBILE CONTROLS -->
<div id="controls">
    <div class="btn" id="up">▲</div>
    <div class="btn" id="left">◀</div>
    <div class="btn" id="right">▶</div>
    <div class="btn" id="down">▼</div>
</div>

<button id="actionButton">ACTION</button>

<div id="cameraText">
    👆 Drag screen<br>
    360° Camera
</div>


<script type="module">

import * as THREE from
'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';

/* ===================================
   BASIC GAME SETUP
=================================== */

const scene = new THREE.Scene();

scene.background = new THREE.Color(0x75c9f5);
scene.fog = new THREE.Fog(0x75c9f5, 80, 260);

const camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
);

const renderer = new THREE.WebGLRenderer({
    antialias: true
});

renderer.setSize(
    window.innerWidth,
    window.innerHeight
);

renderer.shadowMap.enabled = true;

document.getElementById("game")
.appendChild(renderer.domElement);


/* ===================================
   LIGHTS
=================================== */

const sun = new THREE.DirectionalLight(
    0xffffff,
    2
);

sun.position.set(60, 100, 40);
scene.add(sun);

const skyLight = new THREE.HemisphereLight(
    0x87ceeb,
    0x38552b,
    1.5
);

scene.add(skyLight);


/* ===================================
   GROUND
=================================== */

const ground = new THREE.Mesh(

    new THREE.PlaneGeometry(
        500,
        500
    ),

    new THREE.MeshStandardMaterial({
        color: 0x4d913c
    })

);

ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;

scene.add(ground);


/* ===================================
   WATER LAKE
=================================== */

const water = new THREE.Mesh(

    new THREE.PlaneGeometry(110, 65),

    new THREE.MeshStandardMaterial({
        color: 0x218dcc,
        transparent: true,
        opacity: 0.75
    })

);

water.rotation.x = -Math.PI / 2;

water.position.set(
    30,
    0.15,
    -30
);

scene.add(water);


/* ===================================
   PLAYER
=================================== */

const player = new THREE.Group();

const body = new THREE.Mesh(

    new THREE.CylinderGeometry(
        1,
        1.2,
        3,
        8
    ),

    new THREE.MeshStandardMaterial({
        color: 0x151515
    })

);

body.position.y = 2.5;

const head = new THREE.Mesh(

    new THREE.SphereGeometry(
        1,
        16,
        16
    ),

    new THREE.MeshStandardMaterial({
        color: 0xa96d4f
    })

);

head.position.y = 5;

player.add(body);
player.add(head);

scene.add(player);


/* ===================================
   TREES
=================================== */

function createTree(x, z) {

    const tree = new THREE.Group();

    const trunk = new THREE.Mesh(

        new THREE.CylinderGeometry(
            0.4,
            0.7,
            5,
            8
        ),

        new THREE.MeshStandardMaterial({
            color: 0x704214
        })

    );

    trunk.position.y = 2.5;

    const leaves = new THREE.Mesh(

        new THREE.ConeGeometry(
            3.5,
            7,
            10
        ),

        new THREE.MeshStandardMaterial({
            color: 0x196b32
        })

    );

    leaves.position.y = 7;

    tree.add(trunk);
    tree.add(leaves);

    tree.position.set(x, 0, z);

    scene.add(tree);
}


for (let i = 0; i < 100; i++) {

    const x =
        Math.random() * 440 - 220;

    const z =
        Math.random() * 440 - 220;

    if (
        Math.abs(x) < 15 &&
        Math.abs(z) < 15
    ) continue;

    createTree(x, z);
}


/* ===================================
   FOOD COLLECTION
=================================== */

const foods = [];

function createFood(x, z) {

    const food = new THREE.Mesh(

        new THREE.SphereGeometry(
            0.65,
            12,
            12
        ),

        new THREE.MeshStandardMaterial({
            color: 0xff3030
        })

    );

    food.position.set(
        x,
        0.8,
        z
    );

    food.userData.type = "food";

    scene.add(food);

    foods.push(food);
}


for (let i = 0; i < 35; i++) {

    createFood(
        Math.random() * 300 - 150,
        Math.random() * 300 - 150
    );
}


/* ===================================
   WILD ANIMALS
=================================== */

const animals = [];

function createAnimal(x, z) {

    const animal =
        new THREE.Group();

    /* Body */

    const animalBody =
        new THREE.Mesh(

            new THREE.BoxGeometry(
                2.5,
                1.4,
                1.2
            ),

            new THREE.MeshStandardMaterial({
                color: 0x9a643c
            })

        );

    animalBody.position.y = 1.6;

    /* Head */

    const animalHead =
        new THREE.Mesh(

            new THREE.BoxGeometry(
                0.9,
                0.9,
                0.9
            ),

            new THREE.MeshStandardMaterial({
                color: 0xa96d4f
            })

        );

    animalHead.position.set(
        1.5,
        2.1,
        0
    );

    animal.add(animalBody);
    animal.add(animalHead);

    /* Legs */

    for (let i = 0; i < 4; i++) {

        const leg =
            new THREE.Mesh(

                new THREE.BoxGeometry(
                    0.25,
                    1.4,
                    0.25
                ),

                new THREE.MeshStandardMaterial({
                    color: 0x5f3822
                })

            );

        leg.position.set(

            i < 2 ? -0.8 : 0.8,
            0.7,
            i % 2 === 0 ? -0.4 : 0.4

        );

        animal.add(leg);
    }

    animal.position.set(x, 0, z);

    /* Animal movement data */

    animal.userData = {

        direction:
            Math.random() * Math.PI * 2,

        speed:
            0.015 +
            Math.random() * 0.025,

        changeTime:
            Math.random() * 200

    };

    scene.add(animal);

    animals.push(animal);
}


/* Create animals */

for (let i = 0; i < 12; i++) {

    createAnimal(

        Math.random() * 350 - 175,

        Math.random() * 350 - 175

    );
}


/* ===================================
   PLAYER MOVEMENT
=================================== */

const keys = {

    forward: false,
    backward: false,
    left: false,
    right: false

};

const speed = 0.2;


/* Keyboard */

window.addEventListener(
    "keydown",
    e => {

        if (
            e.code === "KeyW" ||
            e.code === "ArrowUp"
        ) keys.forward = true;

        if (
            e.code === "KeyS" ||
            e.code === "ArrowDown"
        ) keys.backward = true;

        if (
            e.code === "KeyA" ||
            e.code === "ArrowLeft"
        ) keys.left = true;

        if (
            e.code === "KeyD" ||
            e.code === "ArrowRight"
        ) keys.right = true;

    }
);

window.addEventListener(
    "keyup",
    e => {

        if (
            e.code === "KeyW" ||
            e.code === "ArrowUp"
        ) keys.forward = false;

        if (
            e.code === "KeyS" ||
            e.code === "ArrowDown"
        ) keys.backward = false;

        if (
            e.code === "KeyA" ||
            e.code === "ArrowLeft"
        ) keys.left = false;

        if (
            e.code === "KeyD" ||
            e.code === "ArrowRight"
        ) keys.right = false;

    }
);


/* Mobile buttons */

function setupButton(id, key) {

    const button =
        document.getElementById(id);

    button.addEventListener(
        "pointerdown",
        e => {

            e.preventDefault();
            keys[key] = true;

        }
    );

    const stop = e => {

        e.preventDefault();
        keys[key] = false;

    };

    button.addEventListener(
        "pointerup",
        stop
    );

    button.addEventListener(
        "pointerleave",
        stop
    );

    button.addEventListener(
        "pointercancel",
        stop
    );

}

setupButton("up", "forward");
setupButton("down", "backward");
setupButton("left", "left");
setupButton("right", "right");


/* ===================================
   360 DEGREE CAMERA
=================================== */

let yaw = 0;
let pitch = 0.25;

let dragging = false;

let oldX = 0;
let oldY = 0;


renderer.domElement.addEventListener(
    "pointerdown",
    e => {

        dragging = true;

        oldX = e.clientX;
        oldY = e.clientY;

    }
);

window.addEventListener(
    "pointerup",
    () => dragging = false
);


window.addEventListener(
    "pointermove",
    e => {

        if (!dragging)
            return;

        const dx =
            e.clientX - oldX;

        const dy =
            e.clientY - oldY;

        oldX = e.clientX;
        oldY = e.clientY;

        yaw -= dx * 0.008;

        pitch -= dy * 0.008;

        pitch = Math.max(
            -0.5,
            Math.min(
                0.8,
                pitch
            )
        );

    }
);


/* ===================================
   SURVIVAL SYSTEM
=================================== */

let health = 100;
let hunger = 100;
let thirst = 100;

let foodCount = 0;

const healthBar =
    document.getElementById("health");

const hungerBar =
    document.getElementById("hunger");

const thirstBar =
    document.getElementById("thirst");

const foodDisplay =
    document.getElementById("foodCount");

const message =
    document.getElementById("message");


function showMessage(text) {

    message.textContent = text;

    clearTimeout(
        window.messageTimer
    );

    window.messageTimer =
        setTimeout(() => {

            message.textContent = "";

        }, 2000);

}


/* Survival decreases */

setInterval(() => {

    hunger = Math.max(
        0,
        hunger - 0.12
    );

    thirst = Math.max(
        0,
        thirst - 0.18
    );

    if (
        hunger <= 0 ||
        thirst <= 0
    ) {

        health = Math.max(
            0,
            health - 0.5
        );

    }

    updateHUD();

}, 1000);


function updateHUD() {

    healthBar.style.width =
        health + "%";

    hungerBar.style.width =
        hunger + "%";

    thirstBar.style.width =
        thirst + "%";

    foodDisplay.textContent =
        foodCount;

}


/* ===================================
   ACTION SYSTEM
=================================== */

function checkAction() {

    /* FOOD COLLECTION */

    for (
        let i = foods.length - 1;
        i >= 0;
        i--
    ) {

        const food =
            foods[i];

        const distance =
            player.position.distanceTo(
                food.position
            );

        if (distance < 4) {

            scene.remove(food);

            foods.splice(i, 1);

            foodCount++;

            foodDisplay.textContent =
                foodCount;

            showMessage(
                "🍎 Food collected!"
            );

            return;
        }

    }


    /* WATER DRINKING */

    const waterCenter =
        new THREE.Vector3(
            water.position.x,
            0,
            water.position.z
        );

    const distanceToWater =
        player.position.distanceTo(
            waterCenter
        );

    if (distanceToWater < 60) {

        thirst = Math.min(
            100,
            thirst + 30
        );

        updateHUD();

        showMessage(
            "💧 Water collected! Thirst +30"
        );

        return;

    }


    /* EAT FOOD */

    if (foodCount > 0) {

        foodCount--;

        hunger = Math.min(
            100,
            hunger + 35
        );

        updateHUD();

        showMessage(
            "🍎 You ate food! Hunger +35"
        );

    } else {

        showMessage(
            "Nothing nearby!"
        );

    }

}


document
.getElementById("actionButton")
.addEventListener(
    "pointerdown",
    e => {

        e.preventDefault();

        checkAction();

    }
);


/* ===================================
   UPDATE ANIMALS
=================================== */

function updateAnimals() {

    animals.forEach(
        animal => {

            const data =
                animal.userData;

            data.changeTime--;

            if (
                data.changeTime <= 0
            ) {

                data.direction =
                    Math.random()
                    * Math.PI * 2;

                data.changeTime =
                    100 +
                    Math.random() * 200;

            }

            animal.position.x +=
                Math.sin(
                    data.direction
                ) *
                data.speed;

            animal.position.z +=
                Math.cos(
                    data.direction
                ) *
                data.speed;

            animal.rotation.y =
                data.direction;

            /* Keep animals inside world */

            if (
                Math.abs(
                    animal.position.x
                ) > 230 ||
                Math.abs(
                    animal.position.z
                ) > 230
            ) {

                data.direction +=
                    Math.PI;

            }

        }
    );

}


/* ===================================
   UPDATE PLAYER
=================================== */

function updatePlayer() {

    const move =
        new THREE.Vector3();

    const forward =
        new THREE.Vector3(

            Math.sin(yaw),
            0,
            Math.cos(yaw)

        );

    const right =
        new THREE.Vector3(

            Math.cos(yaw),
            0,
            -Math.sin(yaw)

        );


    if (keys.forward)
        move.addScaledVector(
            forward,
            -speed
        );

    if (keys.backward)
        move.addScaledVector(
            forward,
            speed
        );

    if (keys.left)
        move.addScaledVector(
            right,
            -speed
        );

    if (keys.right)
        move.addScaledVector(
            right,
            speed
        );


    if (
        move.length() > 0
    ) {

        player.position.add(move);

        player.rotation.y =
            Math.atan2(
                move.x,
                move.z
            );

        body.position.y =
            2.5 +
            Math.sin(
                Date.now() * 0.015
            ) * 0.12;

    }

}


/* ===================================
   UPDATE CAMERA
=================================== */

function updateCamera() {

    const distance = 14;

    camera.position.set(

        player.position.x +

        Math.sin(yaw) *
        Math.cos(pitch) *
        distance,

        player.position.y +
        7 +

        Math.sin(pitch) *
        distance,

        player.position.z +

        Math.cos(yaw) *
        Math.cos(pitch) *
        distance

    );

    camera.lookAt(

        player.position.x,
        player.position.y + 3,
        player.position.z

    );

}


/* ===================================
   GAME LOOP
=================================== */

function animate() {

    requestAnimationFrame(
        animate
    );

    updatePlayer();

    updateAnimals();

    updateCamera();

    /* Water movement */

    water.position.y =
        0.15 +

        Math.sin(
            Date.now() * 0.001
        ) * 0.08;

    renderer.render(
        scene,
        camera
    );

}

animate();


/* ===================================
   RESIZE
=================================== */

window.addEventListener(
    "resize",
    () => {

        camera.aspect =
            window.innerWidth /
            window.innerHeight;

        camera.updateProjectionMatrix();

        renderer.setSize(
            window.innerWidth,
            window.innerHeight
        );

    }
);

updateHUD();

</script>

</body>
</html>
