<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  let canvas;
  let started = false;
  let selected = 'Earth';
  // Desired distance is separate from the live camera distance so travel eases smoothly.
  let desiredCameraDistance = 37;
  let searchQuery = '';
  let activeCatalog = 'ALL';
  let motionPaused = false;
  let showCoordinates = true;
  let scientificMode = true;
  let discoveredObjects = ['Earth'];
  let missionMessage = 'Mission ready. Select an object and run a scan.';
  let constellationMode = false;
  let constellationDepth = false;

  const scienceData = {
    Sun: { distance: 'Solar System centre', fact: 'The Sun contains about 99.86% of the solar system’s mass.', confidence: 'ESTABLISHED' },
    Mercury: { distance: '0.39 AU from Sun (average)', fact: 'Mercury completes one orbit in about 88 Earth days.', confidence: 'ESTABLISHED' },
    Venus: { distance: '0.72 AU from Sun (average)', fact: 'Venus rotates so slowly that one rotation takes longer than its year.', confidence: 'ESTABLISHED' },
    Earth: { distance: '1.00 AU from Sun (average)', fact: 'Earth is the only world currently known to support life.', confidence: 'ESTABLISHED' },
    Mars: { distance: '1.52 AU from Sun (average)', fact: 'Evidence indicates that liquid water flowed across ancient Mars.', confidence: 'ESTABLISHED' },
    Jupiter: { distance: '5.20 AU from Sun (average)', fact: 'Jupiter has the shortest day of any planet in the solar system.', confidence: 'ESTABLISHED' },
    Saturn: { distance: '9.58 AU from Sun (average)', fact: 'Saturn’s rings are made mostly of water-ice particles and rocky debris.', confidence: 'ESTABLISHED' },
    Uranus: { distance: '19.2 AU from Sun (average)', fact: 'Uranus rotates on its side relative to its orbit.', confidence: 'ESTABLISHED' },
    Neptune: { distance: '30.1 AU from Sun (average)', fact: 'Neptune’s discovery was predicted mathematically before it was observed.', confidence: 'ESTABLISHED' },
    'Proxima Centauri': { distance: 'About 4.24 light-years', fact: 'Proxima Centauri is the closest known star to the Sun.', confidence: 'ESTABLISHED' },
    Sirius: { distance: 'About 8.6 light-years', fact: 'Sirius is a binary system; its companion is a faint white dwarf.', confidence: 'ESTABLISHED' },
    'Orion Nebula': { distance: 'About 1,300 light-years', fact: 'The Orion Nebula is an active region of star formation.', confidence: 'ESTABLISHED' },
    'Andromeda Galaxy': { distance: 'About 2.5 million light-years', fact: 'Andromeda is the nearest large spiral galaxy to the Milky Way.', confidence: 'ESTABLISHED' }
  };

  const bodies = [
    { name: 'Sun', color: 0xffb347, size: 2.1, orbit: 0, speed: 0, type: 'STAR', description: 'Our star. Its gravity holds the solar system together.' },
    { name: 'Mercury', color: 0xaaa39a, size: 0.24, orbit: 4, speed: 0.8, type: 'TERRESTRIAL PLANET', description: 'The smallest planet, orbiting closest to the Sun.' },
    { name: 'Venus', color: 0xe6bd83, size: 0.36, orbit: 5.7, speed: 0.58, type: 'TERRESTRIAL PLANET', description: 'A world with a dense atmosphere and extreme surface heat.' },
    { name: 'Earth', color: 0x3c9dff, size: 0.4, orbit: 7.5, speed: 0.42, type: 'TERRESTRIAL PLANET', description: 'Our home world, with liquid water on its surface.' },
    { name: 'Mars', color: 0xd66b49, size: 0.31, orbit: 9.4, speed: 0.34, type: 'TERRESTRIAL PLANET', description: 'The red planet, home to enormous volcanoes and ancient river valleys.' },
    { name: 'Jupiter', color: 0xd8b18a, size: 0.95, orbit: 12.5, speed: 0.19, type: 'GAS GIANT', description: 'The solar system’s largest planet, with a vast family of moons.' },
    { name: 'Saturn', color: 0xe3d1a0, size: 0.78, orbit: 15.8, speed: 0.14, type: 'GAS GIANT', description: 'A gas giant famous for its spectacular rings.' },
    { name: 'Uranus', color: 0x91dce1, size: 0.54, orbit: 19, speed: 0.1, type: 'ICE GIANT', description: 'An ice giant that rotates on a dramatically tilted axis.' },
    { name: 'Neptune', color: 0x426dff, size: 0.52, orbit: 22, speed: 0.075, type: 'ICE GIANT', description: 'A distant ice giant with powerful winds.' }
  ];

  const deepObjects = [
    { name: 'Proxima Centauri', color: 0xff8b62, size: 0.42, position: [-48, 5, -28], type: 'RED DWARF STAR', category: 'DEEP SPACE', description: 'The nearest known star to the Sun, located in the Alpha Centauri system. This scene uses an illustrative position.' },
    { name: 'Sirius', color: 0x9ecbff, size: 0.62, position: [43, 12, -38], type: 'BINARY STAR SYSTEM', category: 'DEEP SPACE', description: 'The brightest star in Earth’s night sky. Its visual placement here is illustrative, not to scale.' },
    { name: 'Orion Nebula', color: 0x8b83ff, size: 5.2, position: [-34, 15, 48], type: 'STAR-FORMING NEBULA', category: 'NEBULA', description: 'A nearby stellar nursery where gas and dust are forming new stars. The glowing cloud is a stylized visualization.' },
    { name: 'Andromeda Galaxy', color: 0x77a9ff, size: 7.5, position: [65, 25, 55], type: 'SPIRAL GALAXY', category: 'GALAXY', description: 'A large spiral galaxy beyond the Milky Way. It is shown as a navigable artistic model, not at true scale.' }
  ];

  $: allObjects = [...bodies, ...deepObjects];
  $: objectInfo = allObjects.find((body) => body.name === selected) || bodies[2];
  $: visibleObjects = allObjects.filter((body) => {
    const matchesQuery = body.name.toLowerCase().includes(searchQuery.toLowerCase());
    const matchesCatalog = activeCatalog === 'ALL' || (activeCatalog === 'SOLAR' ? bodies.some((item) => item.name === body.name) : deepObjects.some((item) => item.name === body.name));
    return matchesQuery && matchesCatalog;
  });

  function selectBody(name) {
    constellationMode = false;
    selected = name;
    const body = allObjects.find((item) => item.name === name);
    if (!body) return;
    if (deepObjects.some((item) => item.name === name)) {
      desiredCameraDistance = name === 'Andromeda Galaxy' ? 14 : name === 'Orion Nebula' ? 10 : 5.5;
    } else {
      // Frame small rocky planets closer and give the giants/Sun more room.
      desiredCameraDistance = name === 'Sun' ? 13 : body.size >= 0.75 ? 6.5 : body.size >= 0.5 ? 5 : 3.8;
    }
  }

  function runScan() {
    if (!discoveredObjects.includes(selected)) discoveredObjects = [...discoveredObjects, selected];
    const data = scienceData[selected];
    missionMessage = data ? `Scan complete: ${selected} added to your discoveries.` : `Scan complete: ${selected} logged.`;
    showCoordinates = true;
  }

  function toggleConstellationView() {
    constellationMode = !constellationMode;
    constellationDepth = false;
    if (constellationMode) {
      desiredCameraDistance = 18;
      missionMessage = 'Orion view opened. Compare its familiar sky pattern with the stars’ different depths.';
    } else {
      desiredCameraDistance = 37;
      missionMessage = 'Returned to the solar-system explorer.';
    }
  }

  function showSystemView() {
    constellationMode = false;
    selected = 'Earth';
    activeCatalog = 'SOLAR';
    searchQuery = '';
    desiredCameraDistance = 37;
    missionMessage = 'System overview restored. Choose a destination to continue.';
  }

  onMount(() => {
    if (!canvas) return;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x020308);
    scene.fog = new THREE.FogExp2(0x020308, 0.0018);

    const camera = new THREE.PerspectiveCamera(55, 1, 0.1, 1200);
    camera.position.set(0, 25, 37);

    const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    canvas.appendChild(renderer.domElement);

    // Keep space dark while allowing the sun-facing sides of planets to read clearly.
    scene.add(new THREE.AmbientLight(0x8b9bbd, 0.22));
    const sunlight = new THREE.PointLight(0xffffff, 1400, 0, 2);
    sunlight.position.set(0, 0, 0);
    scene.add(sunlight);

    // Deep starfield
    const starPositions = [];
    for (let i = 0; i < 3200; i += 1) {
      starPositions.push(
        (Math.random() - 0.5) * 520,
        (Math.random() - 0.5) * 320,
        (Math.random() - 0.5) * 520
      );
    }
    const starGeometry = new THREE.BufferGeometry();
    starGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starPositions, 3));
    const starfield = new THREE.Points(
      starGeometry,
      new THREE.PointsMaterial({
        color: 0xdbe7ff,
        size: 0.34,
        transparent: true,
        opacity: 0.9,
        sizeAttenuation: true
      })
    );
    scene.add(starfield);

    // Distant, stylized deep-space destinations. Positions and sizes are artistic, not to scale.
    const deepMeshes = [];
    deepObjects.forEach((object, index) => {
      const position = new THREE.Vector3(...object.position);
      let mesh;
      if (object.category === 'NEBULA' || object.category === 'GALAXY') {
        const count = object.category === 'GALAXY' ? 1900 : 1250;
        const positions = [];
        const colors = [];
        const base = new THREE.Color(object.color);
        for (let i = 0; i < count; i += 1) {
          const angle = Math.random() * Math.PI * 2;
          const radius = Math.pow(Math.random(), 0.65) * object.size;
          const arm = object.category === 'GALAXY' ? Math.floor(Math.random() * 3) : 0;
          const spiralAngle = angle + (object.category === 'GALAXY' ? radius * 0.75 + arm * (Math.PI * 2 / 3) : 0);
          const spread = object.category === 'GALAXY' ? 0.22 : 0.65;
          positions.push(Math.cos(spiralAngle) * radius, (Math.random() - 0.5) * object.size * spread, Math.sin(spiralAngle) * radius);
          const tint = base.clone().lerp(new THREE.Color(0xdce8ff), Math.random() * 0.55);
          colors.push(tint.r, tint.g, tint.b);
        }
        const geometry = new THREE.BufferGeometry();
        geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
        geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
        const cloud = new THREE.Points(geometry, new THREE.PointsMaterial({ size: object.category === 'GALAXY' ? 0.16 : 0.20, vertexColors: true, transparent: true, opacity: 0.78, depthWrite: false, blending: THREE.AdditiveBlending }));
        cloud.rotation.x = object.category === 'GALAXY' ? 0.45 : 0.18;
        mesh = cloud;
        scene.add(mesh);
        const core = new THREE.Mesh(new THREE.SphereGeometry(object.size * (object.category === 'GALAXY' ? 0.12 : 0.1), 20, 20), new THREE.MeshBasicMaterial({ color: object.color, transparent: true, opacity: 0.24, blending: THREE.AdditiveBlending, depthWrite: false }));
        mesh.add(core);
      } else {
        mesh = new THREE.Mesh(new THREE.SphereGeometry(object.size, 24, 24), new THREE.MeshBasicMaterial({ color: object.color }));
        const glow = new THREE.Mesh(new THREE.SphereGeometry(object.size * 2.5, 20, 20), new THREE.MeshBasicMaterial({ color: object.color, transparent: true, opacity: 0.12, side: THREE.BackSide, blending: THREE.AdditiveBlending, depthWrite: false }));
        mesh.add(glow);
        scene.add(mesh);
      }
      mesh.position.copy(position);
      mesh.userData.objectName = object.name;
      deepMeshes.push({ mesh, object, index });
    });

    // Sun and subtle glow
    const sun = new THREE.Mesh(
      new THREE.SphereGeometry(2.1, 48, 48),
      new THREE.MeshBasicMaterial({ color: 0xffb347 })
    );
    sun.userData.objectName = 'Sun';
    scene.add(sun);

    const sunGlow = new THREE.Mesh(
      new THREE.SphereGeometry(2.8, 32, 32),
      new THREE.MeshBasicMaterial({
        color: 0xff9b45,
        transparent: true,
        opacity: 0.20,
        side: THREE.BackSide,
        blending: THREE.AdditiveBlending,
        depthWrite: false
      })
    );
    sun.add(sunGlow);

    // Planet surface maps. If a remote map fails, the planet keeps its fallback colour.
    const textureLoader = new THREE.TextureLoader();
    /** @type {Record<string, THREE.Texture>} */
    const loadedTextures = {};
    const textureUrls = {
      Mercury: 'https://www.solarsystemscope.com/textures/download/2k_mercury.jpg',
      Venus: 'https://www.solarsystemscope.com/textures/download/2k_venus_surface.jpg',
      Earth: 'https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/textures/planets/earth_atmos_2048.jpg',
      Mars: 'https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/textures/planets/mars_1k_color.jpg',
      Jupiter: 'https://www.solarsystemscope.com/textures/download/2k_jupiter.jpg',
      Saturn: 'https://www.solarsystemscope.com/textures/download/2k_saturn.jpg',
      Uranus: 'https://www.solarsystemscope.com/textures/download/2k_uranus.jpg',
      Neptune: 'https://www.solarsystemscope.com/textures/download/2k_neptune.jpg'
    };

    for (const [name, url] of Object.entries(textureUrls)) {
      const texture = textureLoader.load(
        url,
        () => console.info(`${name} texture loaded`),
        undefined,
        () => console.warn(`${name} texture failed to load; using fallback colour`)
      );
      texture.colorSpace = THREE.SRGBColorSpace;
      loadedTextures[name] = texture;
    }

    const earthCloudTexture = textureLoader.load(
      'https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/textures/planets/earth_clouds_1024.png',
      () => console.info('Earth cloud texture loaded'),
      undefined,
      () => console.warn('Earth cloud texture failed to load')
    );
    earthCloudTexture.colorSpace = THREE.SRGBColorSpace;

    const planetMeshes = [];
    bodies.slice(1).forEach((body, index) => {
      const points = [];
      for (let i = 0; i < 180; i += 1) {
        const angle = (i / 180) * Math.PI * 2;
        points.push(new THREE.Vector3(Math.cos(angle) * body.orbit, 0, Math.sin(angle) * body.orbit));
      }

      const orbit = new THREE.LineLoop(
        new THREE.BufferGeometry().setFromPoints(points),
        new THREE.LineBasicMaterial({ color: 0x27344c, transparent: true, opacity: 0.75 })
      );
      scene.add(orbit);

      const material = new THREE.MeshStandardMaterial({
        color: loadedTextures[body.name] ? 0xffffff : body.color,
        map: loadedTextures[body.name] || null,
        roughness: 0.86,
        metalness: 0.01,
        emissive: body.name === 'Earth' ? 0x07182a : 0x000000,
        emissiveIntensity: body.name === 'Earth' ? 0.04 : 0
      });

      const planet = new THREE.Mesh(
        new THREE.SphereGeometry(body.size, 48, 48),
        material
      );
      const phase = index * 0.72;
      planet.position.set(Math.cos(phase) * body.orbit, 0, Math.sin(phase) * body.orbit);
      planet.userData.objectName = body.name;
      scene.add(planet);

      // A translucent, independently rotating cloud shell around Earth.
      if (body.name === 'Earth') {
        const clouds = new THREE.Mesh(
          new THREE.SphereGeometry(body.size * 1.018, 48, 48),
          new THREE.MeshStandardMaterial({
            map: earthCloudTexture,
            transparent: true,
            opacity: 0.42,
            depthWrite: false,
            blending: THREE.NormalBlending,
            roughness: 1
          })
        );
        clouds.userData.isCloudLayer = true;
        planet.add(clouds);

        // A subtle blue rim, kept separate from the cloud texture.
        const atmosphere = new THREE.Mesh(
          new THREE.SphereGeometry(body.size * 1.055, 48, 48),
          new THREE.MeshBasicMaterial({
            color: 0x398cff,
            transparent: true,
            opacity: 0.09,
            side: THREE.BackSide,
            blending: THREE.AdditiveBlending,
            depthWrite: false
          })
        );
        planet.add(atmosphere);
      }

      planetMeshes.push({ mesh: planet, body, phase });

      if (body.name === 'Saturn') {
        const rings = new THREE.Mesh(
          new THREE.RingGeometry(1.0, 1.55, 80),
          new THREE.MeshBasicMaterial({
            color: 0xc9b58c,
            side: THREE.DoubleSide,
            transparent: true,
            opacity: 0.78
          })
        );
        rings.rotation.x = -Math.PI / 2.6;
        planet.add(rings);
      }
    });

    // Signature discovery: Orion's familiar 2D pattern versus an illustrative depth view.
    // Coordinates are pedagogical, not a precision astronomical reconstruction.
    const constellationRoot = new THREE.Group();
    const skyPattern = new THREE.Group();
    const depthPattern = new THREE.Group();
    const orionStars = [
      { name: 'Betelgeuse', x: -5.2, y: 3.8, z: -2.0, color: 0xffb18a, size: 0.34 },
      { name: 'Bellatrix', x: 4.8, y: 3.2, z: 1.0, color: 0xaecbff, size: 0.27 },
      { name: 'Meissa', x: 0.2, y: 5.2, z: -3.5, color: 0xc4d7ff, size: 0.22 },
      { name: 'Alnitak', x: -1.3, y: 0.2, z: 2.8, color: 0xb9d2ff, size: 0.24 },
      { name: 'Alnilam', x: 0.0, y: 0.0, z: -5.8, color: 0xb9d2ff, size: 0.28 },
      { name: 'Mintaka', x: 1.4, y: -0.2, z: 4.5, color: 0xb9d2ff, size: 0.23 },
      { name: 'Saiph', x: -4.0, y: -4.2, z: 1.5, color: 0xaecbff, size: 0.25 },
      { name: 'Rigel', x: 4.2, y: -4.0, z: -4.0, color: 0x9fc4ff, size: 0.36 }
    ];
    const skyPositions = orionStars.map((star) => new THREE.Vector3(star.x, star.y, 0));
    const depthPositions = orionStars.map((star) => new THREE.Vector3(star.x, star.y, star.z));
    const constellationLinks = [[0, 2], [2, 1], [0, 3], [3, 4], [4, 5], [1, 5], [3, 6], [5, 7]];

    function makeConstellationPattern(group, positions, showDepth) {
      constellationLinks.forEach(([a, b]) => {
        const geometry = new THREE.BufferGeometry().setFromPoints([positions[a], positions[b]]);
        group.add(new THREE.Line(geometry, new THREE.LineBasicMaterial({ color: showDepth ? 0x8fb2ff : 0x7e91bb, transparent: true, opacity: 0.72 })));
      });
      orionStars.forEach((star, index) => {
        const point = new THREE.Mesh(
          new THREE.SphereGeometry(star.size, 18, 18),
          new THREE.MeshBasicMaterial({ color: star.color })
        );
        point.position.copy(positions[index]);
        point.userData.starName = star.name;
        group.add(point);
        const halo = new THREE.Mesh(
          new THREE.SphereGeometry(star.size * 2.5, 12, 12),
          new THREE.MeshBasicMaterial({ color: star.color, transparent: true, opacity: 0.10, blending: THREE.AdditiveBlending, depthWrite: false })
        );
        point.add(halo);
      });
      if (showDepth) {
        const axisMaterial = new THREE.LineDashedMaterial({ color: 0x50658f, dashSize: 0.35, gapSize: 0.22, transparent: true, opacity: 0.45 });
        orionStars.forEach((star, index) => {
          const axis = new THREE.Line(new THREE.BufferGeometry().setFromPoints([skyPositions[index], depthPositions[index]]), axisMaterial);
          axis.computeLineDistances();
          group.add(axis);
        });
      }
    }
    makeConstellationPattern(skyPattern, skyPositions, false);
    makeConstellationPattern(depthPattern, depthPositions, true);
    constellationRoot.add(skyPattern, depthPattern);
    skyPattern.visible = true;
    depthPattern.visible = false;
    constellationRoot.visible = false;
    scene.add(constellationRoot);

    let cameraAngle = 0.35;
    let cameraElevation = 0.58;
    let cameraDistance = 37;
    let dragging = false;
    let lastX = 0;
    let lastY = 0;
    let frameId = 0;
    const raycaster = new THREE.Raycaster();
    raycaster.params.Points.threshold = 1.5;
    const pointer = new THREE.Vector2();
    let pointerDownX = 0;
    let pointerDownY = 0;

    function onPointerDown(event) {
      dragging = true;
      pointerDownX = event.clientX;
      pointerDownY = event.clientY;
      lastX = event.clientX;
      lastY = event.clientY;
      renderer.domElement.setPointerCapture?.(event.pointerId);
    }
    function onPointerUp(event) {
      dragging = false;
      if (!event || Math.hypot(event.clientX - pointerDownX, event.clientY - pointerDownY) > 5) return;
      const rect = renderer.domElement.getBoundingClientRect();
      pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
      raycaster.setFromCamera(pointer, camera);
      const selectable = [sun, ...planetMeshes.map((item) => item.mesh), ...deepMeshes.map((item) => item.mesh)];
      const intersections = raycaster.intersectObjects(selectable, true);
      for (const hit of intersections) {
        let candidate = hit.object;
        while (candidate && !candidate.userData.objectName) candidate = candidate.parent;
        if (candidate?.userData.objectName) {
          selectBody(candidate.userData.objectName);
          break;
        }
      }
    }
    function onPointerMove(event) {
      if (!dragging) return;
      cameraAngle -= (event.clientX - lastX) * 0.005;
      cameraElevation = THREE.MathUtils.clamp(
        cameraElevation + (event.clientY - lastY) * 0.003,
        -1.05,
        1.05
      );
      lastX = event.clientX;
      lastY = event.clientY;
    }
    function onWheel(event) {
      desiredCameraDistance = THREE.MathUtils.clamp(desiredCameraDistance + event.deltaY * 0.025, 3, 100);
    }

    renderer.domElement.addEventListener('pointerdown', onPointerDown);
    window.addEventListener('pointerup', onPointerUp);
    window.addEventListener('pointermove', onPointerMove);
    renderer.domElement.addEventListener('wheel', onWheel, { passive: true });

    function resize() {
      const width = canvas.clientWidth || window.innerWidth;
      const height = canvas.clientHeight || window.innerHeight;
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
    }
    resize();
    window.addEventListener('resize', resize);

    const clock = new THREE.Clock();
    function animate() {
      frameId = requestAnimationFrame(animate);
      const elapsed = clock.getElapsedTime();
      // Ease into the new distance rather than snapping when a world is selected.
      cameraDistance += (desiredCameraDistance - cameraDistance) * 0.035;

      const normalSceneVisible = !constellationMode;
      sun.visible = normalSceneVisible;
      starfield.visible = true;
      planetMeshes.forEach(({ mesh }) => { mesh.visible = normalSceneVisible; });
      deepMeshes.forEach(({ mesh }) => { mesh.visible = normalSceneVisible; });
      scene.children.forEach((child) => {
        if (child.type === 'LineLoop') child.visible = normalSceneVisible;
      });
      if (!motionPaused) {
        planetMeshes.forEach(({ mesh, body, phase }) => {
          const orbitAngle = elapsed * body.speed + phase;
          mesh.position.x = Math.cos(orbitAngle) * body.orbit;
          mesh.position.z = Math.sin(orbitAngle) * body.orbit;
          mesh.rotation.y += 0.003;
          const clouds = mesh.children.find((child) => child.userData.isCloudLayer);
          if (clouds) clouds.rotation.y += 0.0008;
        });
        starfield.rotation.y = elapsed * 0.0015;
        deepMeshes.forEach(({ mesh, object }) => {
          if (object.category === 'GALAXY' || object.category === 'NEBULA') mesh.rotation.y += 0.00012;
        });
      }

      constellationRoot.visible = constellationMode;
      skyPattern.visible = constellationMode && !constellationDepth;
      depthPattern.visible = constellationMode && constellationDepth;
      const selectedBody = allObjects.find((body) => body.name === selected) || bodies[2];
      const selectedMesh = selectedBody.name === 'Sun'
        ? sun
        : (planetMeshes.find((item) => item.body.name === selected)?.mesh || deepMeshes.find((item) => item.object.name === selected)?.mesh || sun);
      const target = constellationMode ? constellationRoot.position : selectedMesh.position;
      const desired = new THREE.Vector3(
        target.x + Math.sin(cameraAngle) * cameraDistance * Math.cos(cameraElevation),
        target.y + Math.sin(cameraElevation) * cameraDistance,
        target.z + Math.cos(cameraAngle) * cameraDistance * Math.cos(cameraElevation)
      );
      camera.position.lerp(desired, 0.025);
      camera.lookAt(target);
      renderer.render(scene, camera);
    }
    animate();

    return () => {
      cancelAnimationFrame(frameId);
      window.removeEventListener('resize', resize);
      window.removeEventListener('pointerup', onPointerUp);
      window.removeEventListener('pointermove', onPointerMove);
      renderer.domElement.removeEventListener('pointerdown', onPointerDown);
      renderer.domElement.removeEventListener('wheel', onWheel);
      renderer.dispose();
      Object.values(loadedTextures).forEach((texture) => texture.dispose());
      earthCloudTexture.dispose();
      scene.traverse((object) => {
        if (object.geometry) object.geometry.dispose();
        if (object.material) {
          if (Array.isArray(object.material)) object.material.forEach((material) => material.dispose());
          else object.material.dispose();
        }
      });
      if (renderer.domElement.parentNode === canvas) canvas.removeChild(renderer.domElement);
    };
  });
</script>

<svelte:head>
  <title>COSMOS — What the Sky Hides</title>
  <meta name="description" content="Explore a stylized 3D universe and discover how a familiar constellation changes when its stars are viewed in depth." />
</svelte:head>

<div class="app-shell">
  <header class="topbar">
    <a class="brand" href="#cosmos" onclick={() => (started = false)}>
      <span class="brand-mark">✳</span>
      <span>COSMOS<span class="brand-dot">.</span></span>
    </a>
    <div class="top-meta"><span class="status-dot"></span> DEEP SPACE EXPLORER</div>
    <button class="top-action" onclick={() => (started = true)}>⌖ &nbsp; EXPLORE UNIVERSE</button>
  </header>

  <!-- Keep the canvas mounted from the beginning; otherwise onMount runs before a conditional canvas exists. -->
  <main class:scene-active={started} class="universe">
    <div class="scene" bind:this={canvas}></div>
    <div class="scene-vignette"></div>

    {#if !started}
      <section class="landing">
        <div class="landing-stars"></div>
        <div class="eyebrow"><span class="status-dot"></span> A WINDOW INTO THE UNKNOWN</div>
        <h1>Somewhere,<br />the universe <span>is waiting.</span></h1>
        <p class="intro-copy">
          From the fire of our nearest star to the quiet edges of our solar system.
          Your journey begins here.
        </p>
        <button class="launch-button" onclick={() => (started = true)}>
          <span class="launch-icon">↗</span> ENTER THE UNIVERSE
        </button>
        <div class="landing-footer"><span>01 / LOCAL SPACE TO DEEP SPACE</span><span>INFINITE QUESTIONS</span></div>
        <div class="hero-orbit orbit-a"></div>
        <div class="hero-orbit orbit-b"></div>
        <div class="hero-planet"></div>
      </section>
    {:else}
      <section class="scene-heading">
        <div class="eyebrow"><span class="status-dot"></span> LIVE SIMULATION</div>
        <h1>{constellationMode ? (constellationDepth ? 'ORION · DEPTH VIEW' : 'ORION · SKY VIEW') : 'THE KNOWN NEIGHBOURHOOD'}</h1>
        <p>{constellationMode ? 'The same constellation, now revealing that its stars sit at different distances.' : 'From our home star to distant stellar nurseries and galaxies.'}</p>
      </section>

      <aside class="object-panel">
        {#if constellationMode}
          <div class="panel-topline"><span>DISCOVERY LAB</span><span class="panel-number">CS / 014</span></div>
          <div class="panel-kicker">SIGNATURE EXPERIMENT</div>
          <h2>Orion’s hidden depth</h2>
          <p class="object-description">Constellations look like flat drawings from Earth. Switch views to see how the stars can sit at very different distances from us.</p>
          <div class="panel-divider"></div>
          <div class="fact-row"><span>CURRENT VIEW</span><strong>{constellationDepth ? '3D DEPTH' : 'SKY PROJECTION'}</strong></div>
          <div class="science-card">
            <div class="science-card-head"><span>WHAT CHANGED?</span><span class="confidence">MODEL</span></div>
            <div class="science-distance">One pattern. Different depths.</div>
            <p>{constellationDepth ? 'The stars have been spread along an illustrative depth axis. Their familiar outline is a perspective effect, not a flat physical arrangement.' : 'This is a simplified sky projection: stars are drawn on one plane to mimic how a constellation appears from Earth.'}</p>
          </div>
          <div class="mission-card">
            <div class="science-card-head"><span>TRY THIS</span><strong>01 / 02</strong></div>
            <p>Toggle between Sky View and Space View. Watch the connecting lines stretch through depth while the familiar pattern changes.</p>
            <button class="scan-button" onclick={() => (constellationDepth = !constellationDepth)}>{constellationDepth ? '◎  RETURN TO SKY VIEW' : '✦  REVEAL STAR DEPTH'}</button>
          </div>
          <div class="panel-divider"></div>
          <button class="scan-button" onclick={toggleConstellationView}>← RETURN TO SPACE EXPLORER</button>
          <p class="disclaimer">Educational visualization only. Star positions and depth offsets are simplified, not a precision 3D star catalogue.</p>
        {:else}
        <div class="panel-topline"><span>OBJECT DATABASE</span><span class="panel-number">CS / 001</span></div>
        <div class="panel-kicker">SELECTED CELESTIAL BODY</div>
        <h2>{objectInfo.name}</h2>
        <p class="object-description">{objectInfo.description}</p>
        <div class="panel-divider"></div>
        <div class="fact-row"><span>OBJECT TYPE</span><strong>{objectInfo.type}</strong></div>
        <div class="fact-row"><span>CATALOG</span><strong>{objectInfo.category || 'SOLAR SYSTEM'}</strong></div>
        <p class="disclaimer">Visual positions and sizes are illustrative, not to scale.</p>
        {#if showCoordinates}
          <div class="scan-readout"><span>SCAN STATUS</span><strong>OBJECT LOCKED</strong><span>TRACKING MODE</span><strong>{motionPaused ? 'STANDBY' : 'LIVE'}</strong></div>
        {/if}
        {#if scientificMode}
          <div class="science-card">
            <div class="science-card-head"><span>SCIENCE BRIEF</span><span class="confidence">{scienceData[objectInfo.name]?.confidence || 'ILLUSTRATIVE'}</span></div>
            <div class="science-distance">{scienceData[objectInfo.name]?.distance || 'Model coordinates only'}</div>
            <p>{scienceData[objectInfo.name]?.fact || 'This object is presented as a stylized visualization; its scene position is not a measured coordinate.'}</p>
          </div>
        {/if}
        <div class="mission-card"><div class="science-card-head"><span>EXPLORATION LOG</span><strong>{discoveredObjects.length.toString().padStart(2, '0')} / {allObjects.length.toString().padStart(2, '0')}</strong></div><div class="mission-progress"><span style={`width: ${(discoveredObjects.length / allObjects.length) * 100}%`}></span></div><p>{missionMessage}</p><button class="scan-button" onclick={runScan}>⌖ &nbsp; RUN OBJECT SCAN</button></div>
        <div class="panel-divider"></div>
        <div class="panel-kicker">OBJECT CATALOG</div>
        <input class="catalog-search" bind:value={searchQuery} placeholder="Search celestial objects…" aria-label="Search celestial objects" />
        <div class="catalog-tabs">
          <button class:tab-active={activeCatalog === 'ALL'} onclick={() => (activeCatalog = 'ALL')}>ALL</button>
          <button class:tab-active={activeCatalog === 'SOLAR'} onclick={() => (activeCatalog = 'SOLAR')}>SOLAR</button>
          <button class:tab-active={activeCatalog === 'DEEP'} onclick={() => (activeCatalog = 'DEEP')}>DEEP SPACE</button>
        </div>
        <div class="object-list">
          {#each visibleObjects as body}
            <button class:active={selected === body.name} class="object-button" onclick={() => selectBody(body.name)}>
              <span class="object-symbol" style={`--planet-color: #${body.color.toString(16).padStart(6, '0')}`}></span>
              <span>{body.name}</span><span class="object-arrow">↗</span>
            </button>
          {/each}
          {#if visibleObjects.length === 0}
            <p class="no-results">No matching objects found.</p>
          {/if}
        </div>
        {/if}
      </aside>

      <div class="scene-controls">
        <div class="control-chip">⤢ &nbsp; DRAG TO ROTATE</div>
        <div class="control-chip">⌕ &nbsp; SCROLL TO ZOOM</div>
        <button class="control-chip control-button" onclick={() => (motionPaused = !motionPaused)}>{motionPaused ? '▶' : 'Ⅱ'} &nbsp; {motionPaused ? 'RESUME MOTION' : 'PAUSE MOTION'}</button>
        <button class="control-chip control-button" onclick={() => (showCoordinates = !showCoordinates)}>{showCoordinates ? '−' : '+'} &nbsp; {showCoordinates ? 'HIDE SCAN DATA' : 'SHOW SCAN DATA'}</button>
        <button class="control-chip control-button" onclick={() => (scientificMode = !scientificMode)}>{scientificMode ? '◉' : '○'} &nbsp; {scientificMode ? 'SCIENCE MODE ON' : 'SCIENCE MODE OFF'}</button>
        <button class="control-chip control-button" onclick={showSystemView}>◎ &nbsp; SYSTEM OVERVIEW</button>
        <button class="control-chip control-button" onclick={toggleConstellationView}>{constellationMode ? '✦ &nbsp; EXIT DISCOVERY LAB' : '✦ &nbsp; ORION DEPTH EXPERIMENT'}</button>
      </div>
      <div class="scene-bottom"><span><span class="status-dot"></span> RENDER ENGINE ONLINE</span><span>SOLAR ORBITS · STYLIZED DEEP-SPACE OBJECTS</span></div>
      <button class="back-button" onclick={() => (started = false)}>← BACK TO INTRO</button>
    {/if}
  </main>
</div>

<style>
  :global(*) { box-sizing: border-box; }
  :global(html), :global(body) { margin: 0; min-height: 100%; background: #020308; color: #f2f5ff; font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; }
  :global(body) { overflow: hidden; }
  :global(button), :global(a) { font: inherit; }
  :global(button) { cursor: pointer; }
  .app-shell { min-height: 100dvh; background: #020308; }
  .topbar { position: absolute; inset: 0 0 auto; height: 74px; padding: 0 5vw; display: flex; align-items: center; justify-content: space-between; z-index: 10; border-bottom: 1px solid #ffffff12; background: #020308a8; backdrop-filter: blur(14px); }
  .brand { display: flex; align-items: center; gap: 10px; color: #f2f5ff; text-decoration: none; letter-spacing: .19em; font-size: 13px; font-weight: 750; }
  .brand-mark { color: #b5c8ff; font-size: 22px; }
  .brand-dot { color: #89a7ff; }
  .top-meta, .eyebrow, .panel-topline, .panel-kicker, .fact-row, .control-chip, .scene-bottom, .landing-footer { font-size: 9px; letter-spacing: .16em; color: #7f8aa6; }
  .top-meta, .eyebrow { display: flex; align-items: center; gap: 9px; }
  .status-dot { display: inline-block; width: 6px; height: 6px; flex: 0 0 auto; border-radius: 50%; background: #8eb5ff; box-shadow: 0 0 12px #83aaffb0; }
  .top-action { background: #b8caff; color: #081020; border: 0; padding: 11px 15px; font-size: 9px; letter-spacing: .1em; font-weight: 800; }
  .landing { position: absolute; inset: 0; overflow: hidden; padding: 22vh 10vw 70px; display: flex; flex-direction: column; align-items: flex-start; justify-content: center; isolation: isolate; background: #020308; z-index: 4; }
  .landing-stars { position: absolute; inset: 0; z-index: -2; background-image: radial-gradient(1px 1px at 12% 19%, #ffffffb0 50%, transparent), radial-gradient(1px 1px at 72% 23%, #b9cfffaa 50%, transparent), radial-gradient(2px 2px at 88% 73%, #ffffffa0 50%, transparent), radial-gradient(1px 1px at 57% 81%, #ffffff90 50%, transparent), radial-gradient(1px 1px at 34% 61%, #b9cfff80 50%, transparent), radial-gradient(1px 1px at 93% 11%, #ffffffa0 50%, transparent); background-size: 370px 300px; }
  .landing h1 { max-width: 780px; margin: 30px 0 20px; font-size: clamp(48px, 7.5vw, 104px); font-weight: 450; letter-spacing: -.065em; line-height: .98; }
  .landing h1 span { color: #8498c9; }
  .intro-copy { max-width: 390px; color: #8d97ad; font-size: 13px; line-height: 1.9; }
  .launch-button { margin-top: 30px; border: 1px solid #93aaff66; background: #9eb6ff; color: #071020; padding: 15px 21px; font-size: 9px; font-weight: 850; letter-spacing: .12em; box-shadow: 0 0 35px #789aff18; }
  .launch-icon { font-size: 15px; margin-right: 14px; }
  .landing-footer { position: absolute; bottom: 28px; left: 5vw; right: 5vw; display: flex; justify-content: space-between; border-top: 1px solid #ffffff19; padding-top: 15px; }
  .hero-planet { position: absolute; width: min(48vw, 550px); aspect-ratio: 1; right: -4%; top: 23%; border-radius: 50%; z-index: -1; background: radial-gradient(circle at 32% 29%, #a6b9de 0%, #596c99 9%, #1c2948 34%, #090e1c 62%, #020308 71%); box-shadow: inset -45px -10px 90px #000b, 0 0 80px #6f8cc91a; }
  .hero-planet::after { content: ""; position: absolute; inset: -10%; border: 1px solid #93aaff20; border-radius: 50%; transform: rotate(-23deg) scaleY(.31); }
  .hero-orbit { position: absolute; right: -10%; top: 31%; width: 60vw; aspect-ratio: 1; border: 1px solid #9ebaff12; border-radius: 50%; z-index: -1; transform: rotate(-25deg) scaleY(.33); }
  .orbit-b { right: -13%; top: 24%; width: 70vw; border-color: #9ebaff0b; }
  .universe { position: fixed; inset: 0; overflow: hidden; background: #020308; }
  .scene { position: absolute; inset: 0; }
  .scene :global(canvas) { display: block; width: 100%; height: 100%; touch-action: none; }
  .scene-vignette { pointer-events: none; position: absolute; inset: 0; background: linear-gradient(90deg, #02030830 0%, transparent 55%, #02030830 100%), linear-gradient(0deg, #02030899 0%, transparent 20%, transparent 70%, #02030865 100%); }
  .scene-heading { pointer-events: none; position: absolute; top: 105px; left: 5vw; z-index: 2; }
  .scene-heading h1 { font-size: 21px; letter-spacing: .18em; font-weight: 520; margin: 15px 0 8px; }
  .scene-heading p { font-size: 11px; color: #828ca3; margin: 0; }
  .object-panel { position: absolute; z-index: 3; top: 100px; right: 3.3vw; bottom: 80px; width: min(285px, 29vw); overflow-y: auto; border: 1px solid #b3c7ff20; background: #070b15ce; backdrop-filter: blur(16px); padding: 22px 20px; }
  .panel-topline { display: flex; justify-content: space-between; font-size: 8px; }
  .panel-number { color: #a1b6e6; }
  .panel-kicker { margin-top: 32px; font-size: 8px; }
  .object-panel h2 { font-size: 32px; letter-spacing: -.04em; font-weight: 480; margin: 10px 0; }
  .object-description { color: #9aa5bb; font-size: 11px; line-height: 1.75; min-height: 57px; }
  .panel-divider { height: 1px; background: #ffffff17; margin: 18px 0; }
  .fact-row { display: flex; justify-content: space-between; align-items: center; margin: 13px 0; font-size: 8px; }
  .fact-row strong { color: #e0e7ff; font-size: 9px; font-weight: 650; letter-spacing: .05em; }
  .disclaimer { color: #69758e; font-size: 9px; line-height: 1.5; }
  .scan-readout { display: grid; grid-template-columns: 1fr auto; gap: 9px 10px; padding: 12px 0 2px; font-size: 8px; letter-spacing: .11em; color: #7785a2; }
  .scan-readout strong { color: #b7c8f5; font-size: 8px; font-weight: 650; text-align: right; }
  .science-card, .mission-card { margin-top: 15px; padding: 12px; border: 1px solid #9bb5ff25; background: linear-gradient(135deg, #9bb5ff0d, #ffffff03); }
  .science-card-head { display: flex; align-items: center; justify-content: space-between; gap: 8px; color: #8c9cbd; font-size: 7px; letter-spacing: .13em; }
  .confidence { color: #8bcab5; font-size: 7px; text-align: right; }
  .science-distance { margin-top: 10px; color: #e0e8ff; font-size: 11px; font-weight: 650; letter-spacing: -.01em; }
  .science-card p, .mission-card p { color: #8f9bb4; font-size: 9px; line-height: 1.65; margin: 8px 0 0; }
  .mission-card { margin-top: 9px; }
  .mission-card strong { color: #c9d5ff; font-size: 9px; }
  .mission-progress { height: 3px; margin-top: 10px; background: #ffffff12; overflow: hidden; }
  .mission-progress span { display: block; height: 100%; background: #9eb6ff; transition: width .25s ease; }
  .scan-button { width: 100%; margin-top: 11px; padding: 9px 8px; border: 1px solid #9bb5ff40; background: #9bb5ff12; color: #dbe4ff; font-size: 8px; letter-spacing: .1em; }
  .scan-button:hover { background: #9bb5ff25; border-color: #9bb5ff80; }
  .catalog-search { width: 100%; margin-top: 12px; padding: 10px 11px; border: 1px solid #ffffff20; outline: none; background: #ffffff08; color: #edf2ff; font-size: 10px; }
  .catalog-search:focus { border-color: #9bb5ff80; }
  .catalog-search::placeholder { color: #68748d; }
  .catalog-tabs { display: grid; grid-template-columns: repeat(3, 1fr); gap: 4px; margin-top: 7px; }
  .catalog-tabs button { padding: 7px 3px; border: 1px solid #ffffff16; background: #ffffff05; color: #78849c; font-size: 7px; letter-spacing: .08em; }
  .catalog-tabs button.tab-active { color: #eaf0ff; border-color: #9bb5ff58; background: #9bb5ff14; }
  .no-results { grid-column: 1 / -1; color: #8792a8; font-size: 10px; line-height: 1.5; }
  .object-list { display: grid; grid-template-columns: 1fr 1fr; gap: 5px; margin-top: 10px; }
  .object-button { display: flex; align-items: center; gap: 7px; min-width: 0; padding: 9px 7px; border: 1px solid transparent; color: #8d99b2; background: #ffffff05; text-align: left; font-size: 10px; }
  .object-button:hover, .object-button.active { color: #eaf0ff; border-color: #9bb5ff48; background: #9bb5ff12; }
  .object-symbol { width: 7px; height: 7px; flex: 0 0 auto; border-radius: 50%; background: var(--planet-color); box-shadow: 0 0 8px var(--planet-color); }
  .object-arrow { margin-left: auto; color: #7f9cff; font-size: 11px; }
  .scene-controls { position: absolute; z-index: 3; bottom: 65px; left: 5vw; display: flex; gap: 8px; }
  .control-chip { padding: 11px 12px; border: 1px solid #ffffff20; background: #050914bd; color: #a3aec5; font-size: 8px; }
  .control-button { cursor: pointer; font: inherit; text-align: left; letter-spacing: inherit; }
  .control-button:focus-visible, .object-button:focus-visible, .scan-button:focus-visible, .launch-button:focus-visible, .top-action:focus-visible, .back-button:focus-visible { outline: 2px solid #b8caff; outline-offset: 3px; }
  .control-button:hover { border-color: #9bb5ff70; color: #eaf0ff; background: #9bb5ff12; }
  .scene-bottom { position: absolute; z-index: 3; bottom: 24px; left: 5vw; right: 5vw; display: flex; justify-content: space-between; font-size: 8px; }
  .scene-bottom .status-dot { margin-right: 8px; }
  .back-button { position: absolute; z-index: 4; top: 88px; left: 5vw; border: 1px solid #ffffff25; background: #050914bd; color: #a9b5cc; padding: 9px 12px; font-size: 8px; letter-spacing: .1em; }
  @media (max-width: 700px) {
    .topbar { height: 62px; padding: 0 18px; }
    .top-meta { display: none; }
    .top-action { padding: 9px; font-size: 8px; }
    .landing { padding: 20vh 7vw 70px; }
    .landing h1 { font-size: clamp(45px, 13vw, 70px); }
    .hero-planet { width: 75vw; right: -28%; top: 18%; opacity: .6; }
    .scene-heading { top: 83px; left: 20px; }
    .scene-heading h1 { font-size: 16px; }
    .object-panel { top: auto; bottom: 80px; right: 12px; width: min(255px, 68vw); max-height: 55vh; padding: 16px; }
    .panel-kicker { margin-top: 18px; }
    .object-panel h2 { font-size: 26px; }
    .science-card, .mission-card { padding: 10px; }
    .scene-controls { max-width: 46vw; flex-wrap: wrap; }
    .object-list { grid-template-columns: 1fr 1fr; }
    .scene-controls { left: 12px; bottom: 24px; flex-direction: column; gap: 4px; max-width: 43vw; max-height: 47vh; overflow-y: auto; }
    .control-chip { padding: 8px; font-size: 7px; }
    .scene-controls .control-chip { white-space: nowrap; }
    .scene-bottom { display: none; }
    .back-button { top: 69px; left: 20px; }
  }
</style>
