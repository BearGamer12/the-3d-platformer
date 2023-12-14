<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Platformer</title>
    <script src="https://threejs.org/build/three.js"></script>
</head>

<body style="margin: 0; overflow: hidden;">

    <div id="start-menu" style="text-align: center; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);">
        <h1 style="font-size: 5vw; margin-bottom: 2vw;">Welcome to the 3D Platformer</h1>
        <button onclick="startGame()" style="width: 40vw; height: 20vw; font-size: 2vw;">Start Game</button>
        <button onclick="toggleLevelEditor()" style="width: 40vw; height: 20vw; font-size: 2vw;">Toggle Level Editor</button>
    </div>

    <div id="game-container" style="display: none;"></div>

    <!-- Level Editor UI -->
    <div id="level-editor-ui" style="display: none; position: absolute; top: 10px; left: 10px; z-index: 1;">
        <label for="platform-height-slider">Platform Height:</label>
        <input type="range" id="platform-height-slider" min="1" max="10" value="5" step="1" oninput="updatePlatformHeight()">
        <span id="current-height">5</span>

        <br>

        <label for="platform-x-position-slider">Platform X Position:</label>
        <input type="range" id="platform-x-position-slider" min="-10" max="10" value="0" step="0.1" oninput="updatePlatformPosition('x')">
        <span id="current-x-position">0</span>

        <br>

        <label for="platform-y-position-slider">Platform Y Position:</label>
        <input type="range" id="platform-y-position-slider" min="-10" max="10" value="0" step="0.1" oninput="updatePlatformPosition('y')">
        <span id="current-y-position">0</span>

        <br>

        <label for="platform-z-position-slider">Platform Z Position:</label>
        <input type="range" id="platform-z-position-slider" min="-50" max="50" value="0" step="0.1" oninput="updatePlatformPosition('z')">
        <span id="current-z-position">0</span>
    </div>

    <script>
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer();
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('game-container').appendChild(renderer.domElement);

        // Set the scene background to sky blue
        scene.background = new THREE.Color(0x87CEEB); // Sky blue color

        const characterGeometry = new THREE.BoxGeometry(1, 1, 1);
        const characterMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
        const character = new THREE.Mesh(characterGeometry, characterMaterial);
        character.position.y = 2;
        const initialPosition = new THREE.Vector3();
        initialPosition.copy(character.position);
        scene.add(character);

        const platformGeometry = new THREE.BoxGeometry(5, 1, 5);
        const platformMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00 });

        const platforms = [];

        // Function to create initial platforms
        function createInitialPlatforms() {
            const initialPlatforms = [
                new THREE.Mesh(platformGeometry, platformMaterial),
                new THREE.Mesh(platformGeometry, platformMaterial),
                new THREE.Mesh(platformGeometry, platformMaterial),
                new THREE.Mesh(platformGeometry, platformMaterial),
                new THREE.Mesh(platformGeometry, platformMaterial),
                new THREE.Mesh(platformGeometry, platformMaterial),
                new THREE.Mesh(platformGeometry, platformMaterial),
            ];

            initialPlatforms[0].position.y = 0;
            initialPlatforms[1].position.set(0, 3, -5);
            initialPlatforms[2].position.set(-3, 3, -15);
            initialPlatforms[3].position.set(3, 3, -25);
            initialPlatforms[4].position.set(0, 3, -35);
            initialPlatforms[5].position.set(-3, 3, -45);
            initialPlatforms[6].position.set(3, 2.5, -55);

            for (const platform of initialPlatforms) {
                scene.add(platform);
                platforms.push(platform);
            }
        }

        createInitialPlatforms();

        camera.position.y = 5;
        camera.position.z = 8;

        const characterState = {
            position: { x: 0, y: 2, z: 0 },
            velocity: { x: 0, y: 0, z: 0 },
            jumping: false,
        };

        let belowTeleportThreshold = false;
        let isLevelEditorActive = false;

        const handleKeyDown = (event) => {
            if (event.key === 'ArrowUp') {
                characterState.velocity.z = -0.1;
            } else if (event.key === 'ArrowDown') {
                characterState.velocity.z = 0.1;
            } else if (event.key === 'ArrowLeft') {
                characterState.velocity.x = -0.1;
            } else if (event.key === 'ArrowRight') {
                characterState.velocity.x = 0.1;
            }

            if (event.key === ' ' && !characterState.jumping) {
                characterState.velocity.y = 0.2;
                characterState.jumping = true;
            }
        };

        const handleKeyUp = (event) => {
            if (event.key === 'ArrowUp' || event.key === 'ArrowDown') {
                characterState.velocity.z = 0;
            } else if (event.key === 'ArrowLeft' || event.key === 'ArrowRight') {
                characterState.velocity.x = 0;
            }
        };

        window.addEventListener('keydown', handleKeyDown);
        window.addEventListener('keyup', handleKeyUp);

        function startGame() {
            document.getElementById('start-menu').style.display = 'none';
            document.getElementById('game-container').style.display = 'block';
            update();
        }

        function toggleLevelEditor() {
            isLevelEditorActive = !isLevelEditorActive;

            if (isLevelEditorActive) {
                createBasePlatform();
                document.getElementById('level-editor-ui').style.display = 'block';
                document.addEventListener('click', handleLevelEditorClick);
            } else {
                document.getElementById('level-editor-ui').style.display = 'none';
                document.removeEventListener('click', handleLevelEditorClick);
            }
        }

        function handleLevelEditorClick(event) {
            if (isLevelEditorActive) {
                const mouse = new THREE.Vector2();
                mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
                mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

                const raycaster = new THREE.Raycaster();
                raycaster.setFromCamera(mouse, camera);

                const intersection = new THREE.Vector3();
                raycaster.ray.intersectPlane(new THREE.Plane(new THREE.Vector3(0, 1, 0)), intersection);

                createBasePlatform(intersection);
            }
        }

        function createBasePlatform(position = new THREE.Vector3(0, 0, 0)) {
            const newPlatform = new THREE.Mesh(platformGeometry, platformMaterial);
            newPlatform.position.copy(position);
            scene.add(newPlatform);
            platforms.push(newPlatform);
        }

        const update = function () {
            characterState.velocity.y -= 0.01;

            characterState.position.x += characterState.velocity.x;
            characterState.position.y += characterState.velocity.y;
            characterState.position.z += characterState.velocity.z;

            for (const platform of platforms) {
                const characterBox = new THREE.Box3().setFromObject(character);
                const platformBox = new THREE.Box3().setFromObject(platform);

                if (characterBox.intersectsBox(platformBox) && characterState.velocity.y < 0) {
                    characterState.position.y = platform.position.y + platform.geometry.parameters.height / 2 + character.geometry.parameters.height / 2;
                    characterState.velocity.y = 0;
                    characterState.jumping = false;
                    belowTeleportThreshold = false;
                }
            }

            if (characterState.position.y < -10 && !belowTeleportThreshold) {
                document.getElementById('start-menu').style.display = 'block';
                document.getElementById('game-container').style.display = 'none';
                characterState.position.copy(initialPosition);
                characterState.velocity.set(0, 0, 0);
                belowTeleportThreshold = true;
            }

            camera.position.copy(characterState.position);
            camera.position.y += 5;
            camera.position.z += 8;

            character.position.copy(characterState.position);

            renderer.render(scene, camera);

            requestAnimationFrame(update);
        };

        window.addEventListener('resize', function () {
            const newWidth = window.innerWidth;
            const newHeight = window.innerHeight;

            camera.aspect = newWidth / newHeight;
            camera.updateProjectionMatrix();

            renderer.setSize(newWidth, newHeight);
        });

        // Function to update platform height
        window.updatePlatformHeight = function () {
            const sliderValue = document.getElementById('platform-height-slider').value;
            document.getElementById('current-height').innerText = sliderValue;

            const selectedPlatform = platforms[platforms.length - 1]; // Select the last created platform
            selectedPlatform.scale.y = sliderValue;
        };

        // Function to update platform position
        window.updatePlatformPosition = function (axis) {
            const sliderValue = parseFloat(document.getElementById(`platform-${axis}-position-slider`).value);
            document.getElementById(`current-${axis}-position`).innerText = sliderValue;

            const selectedPlatform = platforms[platforms.length - 1]; // Select the last created platform

            switch (axis) {
                case 'x':
                    selectedPlatform.position.x = sliderValue;
                    break;
                case 'y':
                    selectedPlatform.position.y = sliderValue;
                    break;
                case 'z':
                    selectedPlatform.position.z = sliderValue;
                    break;
                default:
                    break;
            }
        };

        update();
    </script>

</body>

</html>
