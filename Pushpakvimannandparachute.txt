// ============================================================================
// PUSHPAKA VIMANA — Crash-Free & Mobile-Optimized Interactive 3D Sketch (APDE)
// Key Fixes & Optimizations:
//  - Zero per-frame PShape.setVertex() mutation (immutable retained VBOs only)
//  - Perspective projection restored every frame after ortho() HUD pass
//  - Explicit 7-light golden-hour rig (strictly <= PGL.MAX_LIGHTS = 8)
//  - Safe landscape settle gate & 11-step progressive build + GPU VBO warm-up
//  - Sub-15,000-vertex VBO chunks (< 16,384 heap step & 32,767 short index limit)
//  - ~46% vertex reduction & lower cloud overdraw with identical visual output
// ============================================================================

import java.util.HashMap;

// Procedural Textures
PImage texUpperPetal;
PImage texDomeFiligree;
PImage texLowerRotor;
PImage texDeckFloor;
PImage texCloud;
PImage texManArmor;
PImage texBlurHalo;

// Immutable Retained 3D Meshes (Every PShape strictly <= 15,000 vertices)
PShape shapeSkyDome;
PShape shapeTerrain;
PShape shapeDeckSlabsAndCeiling;         // Merged Upper + Lower Deck + Ceiling Disc
PShape shapeColumnAndDome;               // Merged Central Axis Column + Kalasha Dome
MultiMesh meshUpperDeckStatic;           // Auto-chunked Upper Deck pillars, rails, banners, ribs
MultiMesh meshLowerDeckAndFinialStatic;  // Auto-chunked Lower Deck pillars, rails, rings, crown
MultiMesh meshAllClosedPetals;           // 24 closed curved lotus petals (1 chunk)
PShape shapeRotorHub;
MultiMesh meshAllRotorBlades;            // 32 rotor lotus blades (1 chunk)

// Woman 3D Meshes (1.68m human scale — standing on Lower Deck at y = -4.20m)
MultiMesh meshWomanBodyStatic;
MultiMesh meshWomanHeadFull;

// Warrior Man 3D Meshes (1.84m human scale — standing on Upper Deck at y = 0.0m)
MultiMesh meshManBodyStatic;
PShape shapeManArmorTorso;
MultiMesh meshManHeadFull;

// Sunlit Cumulus Cloud Layer (Immediate-mode textured batch; zero retessellation)
final int CLOUD_COUNT = 72;
CloudPuff[] clouds = new CloudPuff[CLOUD_COUNT];

// Two-Tier Deck Vertical Elevations (World space: +Y is UP)
final float UPPER_DECK_Y = 0.0f;
final float LOWER_DECK_Y = -4.20f;

// Woman Placement on Lower Hollow Colonnade Deck (y = -4.20m, r = 3.60m to 8.40m)
final float WOMAN_ANGLE = 0.48f;
final float WOMAN_RADIUS = 4.55f;
float womanX, womanZ;

// Warrior Man Placement on Upper Hollow Colonnade Deck (y = 0.0m, r = 4.20m to 9.50m)
final float MAN_ANGLE = 0.34f;
final float MAN_RADIUS = 5.35f;
float manX, manZ;

// Camera & Orbit State (World space: +Y is UP, converted via -Y in P3D)
PVector camPos = new PVector();
PVector camTarget = new PVector();
PVector goalCamPos = new PVector();
PVector goalCamTarget = new PVector();
PVector[] presetPositions = new PVector[4];
PVector[] presetTargets = new PVector[4];
String[] presetLabels = {
  "1. Man Above & Woman Below",
  "2. Aerial Lotus Vimana",
  "3. Woman & Lower Deck",
  "4. Closed Petals & Rotor"
};

// Reusable Scratch Vectors & Precomputed Constants
PVector scratchGoalPos = new PVector();
PVector scratchGoalLook = new PVector();
PVector scratchCamFwd = new PVector();
PVector scratchCamRight = new PVector();
PVector sunDirLight = new PVector();
PVector bounceDirLight = new PVector();
final float CAM_FOV = radians(46.0f);

int activePreset = 0;
boolean isTransitioningCamera = false;
float camDistance = 12.0f;
float camYaw = 0.85f;
float camPitch = 0.24f;
float prevHoverY = 0.0f;
int lastViewportW = -1;
int lastViewportH = -1;
float cachedAspect = 16.0f / 9.0f;

// Rotor & Simulation State
float rotorAngle = 0.0f;
float rotorSpeed = 15.0f; // Default 15.0 rad/s (~143 RPM)
float lastFormattedRotorSpeed = -1.0f;
String cachedRpmLabel = "15.0 rad/s (143 RPM)";
boolean isSlowMo = false;
float terrainScrollZ = 0.0f;
long lastMillis = 0;
float elapsedTime = 0.0f;

// Progressive Loading & Lifecycle State
boolean isInitialized = false;
int initStep = 0;
int orientWaitFrames = 0;
String loadingStatus = "Preparing golden-hour 3D stage...";
boolean uiVisible = true;
boolean draggingSlider = false;
boolean draggingOrbit = false;
boolean wasPinching = false;
float prevTouchX = 0, prevTouchY = 0;
float prevPinchDist = -1;
PVector prevPinchMid = new PVector();

// Cached HUD Layout & Palette
int hudLayoutW = -1;
int hudLayoutH = -1;
float hudSc, hudPad, hudTitleW, hudTitleH;
float hudSubTextSize, hudPresetTextSize;
float hudRotorPanelW, hudRotorPanelH, hudRotorX, hudRotorY;
float hudSliderX, hudSliderW, hudSliderY;
float hudCamPanelW, hudCamPanelH, hudCamPanelX, hudCamPanelY;
float[] rectHideBtn = new float[4];
float[][] rectPresetBtns = new float[4][4];
float[] rectSliderTrack = new float[4];
float[] rectSlowMoBtn = new float[4];

int colHudGoldText, colHudCreamText;
int colBtnActiveFill, colBtnInactiveFill, colBtnActiveStroke, colBtnInactiveStroke;
int colSlowMoOnFill, colSlowMoOffFill, colSliderTrack, colSliderFill;

// ============================================================================
// 1. SETTINGS & SETUP
// ============================================================================
void settings() {
  fullScreen(P3D);
}

void setup() {
  orientation(LANDSCAPE);
  frameRate(60);

  sunDirLight.set(-68.0f, 42.0f, 62.0f).normalize();
  bounceDirLight.set(40.0f, -35.0f, -45.0f).normalize();

  womanX = cos(WOMAN_ANGLE) * WOMAN_RADIUS;
  womanZ = sin(WOMAN_ANGLE) * WOMAN_RADIUS;

  manX = cos(MAN_ANGLE) * MAN_RADIUS;
  manZ = sin(MAN_ANGLE) * MAN_RADIUS;

  // Define the 4 Camera Presets
  presetPositions[0] = new PVector(13.8f, 2.35f, 8.9f);
  presetTargets[0]   = new PVector(3.55f, -1.25f, 1.55f);

  presetPositions[1] = new PVector(31.5f, 25.0f, 34.0f);
  presetTargets[1]   = new PVector(0.0f, 3.6f, 0.0f);

  presetPositions[2] = new PVector(8.85f, -2.05f, 6.45f);
  presetTargets[2]   = new PVector(2.85f, -3.35f, 1.65f);

  presetPositions[3] = new PVector(26.5f, -6.2f, 28.5f);
  presetTargets[3]   = new PVector(0.0f, -1.8f, 0.0f);

  camPos.set(presetPositions[0]);
  camTarget.set(presetTargets[0]);
  goalCamPos.set(presetPositions[0]);
  goalCamTarget.set(presetTargets[0]);
  syncSphericalFromCartesian();

  // Precompute HUD colors once
  colHudGoldText       = color(253, 230, 168);
  colHudCreamText      = color(254, 243, 199);
  colBtnActiveFill     = color(235, 178, 68, 95);
  colBtnInactiveFill   = color(255, 255, 255, 16);
  colBtnActiveStroke   = color(255, 216, 128, 220);
  colBtnInactiveStroke = color(251, 191, 36, 70);
  colSlowMoOnFill      = color(245, 158, 11, 120);
  colSlowMoOffFill     = color(245, 158, 11, 55);
  colSliderTrack       = color(255, 215, 130, 55);
  colSliderFill        = color(245, 185, 66, 200);

  lastMillis = millis();
}

// Progressive multi-step initialization & GPU VBO warm-up prevents ANR & spikes
void runInitializationStep() {
  // Wait for orientation(LANDSCAPE) resize to settle before allocating GL assets
  if (width < height && orientWaitFrames < 18) {
    orientWaitFrames++;
    return;
  }

  if (initStep == 0) {
    loadingStatus = "Weaving procedural lotus & dome filigree textures...";
    texUpperPetal = createUpperLotusPetalTexture();
    texDomeFiligree = createDomeFiligreeTexture();
    texLowerRotor = createLowerRotorBladeTexture();
    initStep++;
  } else if (initStep == 1) {
    loadingStatus = "Crafting deck mandala, cloud & armor textures...";
    texDeckFloor = createDeckFloorTexture();
    texCloud = createCloudTexture();
    texManArmor = createManArmorTexture();
    texBlurHalo = createBlurHaloTexture();
    initStep++;
  } else if (initStep == 2) {
    loadingStatus = "Building golden-hour sky & mountain valleys...";
    shapeSkyDome = buildSkyDome(950.0f, 24, 12);
    shapeTerrain = buildTerrainMesh(1400.0f, 48); // 48x48x6 = 13,824 verts (< 16,384)
    initClouds();
    initStep++;
  } else if (initStep == 3) {
    loadingStatus = "Constructing deck slabs, axis column & Kalasha dome...";
    buildDeckSlabsAndDomeShapes();
    initStep++;
  } else if (initStep == 4) {
    loadingStatus = "Sculpting upper hollow colonnade deck...";
    buildUpperDeckStaticShapes();
    initStep++;
  } else if (initStep == 5) {
    loadingStatus = "Sculpting lower colonnade & crown finial...";
    buildLowerDeckAndFinialStaticShapes();
    initStep++;
  } else if (initStep == 6) {
    loadingStatus = "Sculpting 24 closed petals & 32 rotor blades...";
    buildLotusPetalsAndRotorShapes();
    initStep++;
  } else if (initStep == 7) {
    loadingStatus = "Sculpting Woman figure for lower deck...";
    buildWomanShapes();
    initStep++;
  } else if (initStep == 8) {
    loadingStatus = "Sculpting Warrior Man & Kavacha armor...";
    buildManShapes();
    initStep++;
  } else if (initStep == 9) {
    loadingStatus = "Uploading architecture VBOs to mobile GPU (1/2)...";
    initStep++;
  } else if (initStep == 10) {
    loadingStatus = "Uploading figure & rotor VBOs to mobile GPU (2/2)...";
    lastMillis = millis();
    isInitialized = true;
  }
}

void beginWarmupPass() {
  float aspect = (float) width / max((float) height, 1.0f);
  perspective(CAM_FOV, aspect, 0.12f, 1800.0f);
  camera(0.0f, 0.0f, 10.0f, 0.0f, 0.0f, 0.0f, 0.0f, 1.0f, 0.0f);
  setupSceneLights(0.0f);
  pushMatrix();
  translate(0.0f, 0.0f, -5000.0f);
}

void endWarmupPass() {
  popMatrix();
  noLights();
}

void warmUpGpuBatch1() {
  beginWarmupPass();
  shape(shapeSkyDome);
  shape(shapeTerrain);
  shape(shapeDeckSlabsAndCeiling);
  shape(shapeColumnAndDome);
  meshUpperDeckStatic.draw();

  // Warm up immediate-mode texture uploads (texCloud & texBlurHalo)
  noStroke();
  textureMode(NORMAL);
  beginShape(TRIANGLES);
  texture(texCloud);
  vertex(-1, -1, 0, 0, 0);
  vertex( 1, -1, 0, 1, 0);
  vertex( 1,  1, 0, 1, 1);
  endShape();

  beginShape(TRIANGLES);
  texture(texBlurHalo);
  vertex(-1, -1, 0, 0, 0);
  vertex( 1, -1, 0, 1, 0);
  vertex( 1,  1, 0, 1, 1);
  endShape();
  endWarmupPass();
}

void warmUpGpuBatch2() {
  beginWarmupPass();
  meshLowerDeckAndFinialStatic.draw();
  meshAllClosedPetals.draw();
  shape(shapeRotorHub);
  meshAllRotorBlades.draw();
  meshWomanBodyStatic.draw();
  meshWomanHeadFull.draw();
  meshManBodyStatic.draw();
  shape(shapeManArmorTorso);
  meshManHeadFull.draw();
  endWarmupPass();
}

// ============================================================================
// 2. MAIN DRAW LOOP
// ============================================================================
void draw() {
  if (!isInitialized) {
    if (initStep == 9) warmUpGpuBatch1();
    else if (initStep == 10) warmUpGpuBatch2();
    drawLoadingScreen();
    runInitializationStep();
    return;
  }

  long now = millis();
  float dt = (now - lastMillis) * 0.001f;
  lastMillis = now;
  if (dt < 0.0f || dt > 0.1f) dt = 0.016f;
  else dt = min(dt, 0.05f);
  elapsedTime += dt;

  // Update cached aspect when viewport changes; ALWAYS restore perspective() after HUD ortho()
  if (width != lastViewportW || height != lastViewportH) {
    lastViewportW = width;
    lastViewportH = height;
    cachedAspect = (float) width / max((float) height, 1.0f);
  }
  perspective(CAM_FOV, cachedAspect, 0.12f, 1800.0f);

  // 1. Update Rotor Rotation & Gentle Hover Buoyancy
  rotorAngle = (rotorAngle + rotorSpeed * dt) % TWO_PI;
  float hoverY = sin(elapsedTime * 1.15f) * 0.28f;
  float deltaHoverY = hoverY - prevHoverY;
  prevHoverY = hoverY;

  camPos.y += deltaHoverY;
  camTarget.y += deltaHoverY;

  // 2. Smooth Camera Preset Interpolation
  if (isTransitioningCamera) {
    scratchGoalPos.set(goalCamPos.x, goalCamPos.y + hoverY, goalCamPos.z);
    scratchGoalLook.set(goalCamTarget.x, goalCamTarget.y + hoverY, goalCamTarget.z);

    float alpha = 1.0f - exp(-4.5f * dt);
    camPos.lerp(scratchGoalPos, alpha);
    camTarget.lerp(scratchGoalLook, alpha);
    syncSphericalFromCartesian();

    if (camPos.dist(scratchGoalPos) < 0.04f && camTarget.dist(scratchGoalLook) < 0.04f) {
      isTransitioningCamera = false;
    }
  }

  // Apply Camera (negate Y so +Y in world coordinates points UP on screen)
  camera(
    camPos.x, -camPos.y, camPos.z,
    camTarget.x, -camTarget.y, camTarget.z,
    0.0f, 1.0f, 0.0f
  );

  background(40, 75, 122);

  // 3. Render Golden-Hour Sky Dome (unlit, depth write disabled)
  noLights();
  hint(DISABLE_DEPTH_MASK);
  pushMatrix();
  translate(camPos.x, -camPos.y, camPos.z);
  shape(shapeSkyDome);
  popMatrix();
  hint(ENABLE_DEPTH_MASK);

  // 4. Golden-Hour Atmospheric & Dual-Deck Colonnade Lighting (7 lights <= MAX_LIGHTS 8)
  setupSceneLights(hoverY);

  // 5. Scrolling Lush Mountain Valley Terrain Below (y = -118m)
  float flightSpeed = 14.0f + rotorSpeed * 0.65f;
  terrainScrollZ += flightSpeed * 0.42f * dt;
  if (terrainScrollZ > 220.0f) terrainScrollZ -= 440.0f;

  pushMatrix();
  translate(0.0f, 118.0f, terrainScrollZ);
  shape(shapeTerrain);
  popMatrix();

  // 6. Render the Pushpaka Vimana Root Hierarchy
  pushMatrix();
  translate(0.0f, -hoverY, 0.0f);
  rotateZ(sin(elapsedTime * 0.72f) * 0.012f);
  rotateX(cos(elapsedTime * 0.65f) * 0.010f);

  // 6A. Merged Deck Slabs & Ceiling, Central Column & Kalasha Dome, Static Architecture
  specular(255, 226, 150);
  shininess(28.0f);
  shape(shapeDeckSlabsAndCeiling);

  specular(255, 232, 165);
  shininess(36.0f);
  shape(shapeColumnAndDome);

  meshUpperDeckStatic.draw();
  meshLowerDeckAndFinialStatic.draw();

  // 6B. Warrior Man (1.84m) Standing Above on Upper Deck (y = 0.0m)
  drawMan();

  // 6C. Woman (1.68m) Standing Below on Lower Deck (y = -4.20m)
  drawWoman();

  // 6D. 24 Closed Curved Lotus Petals Cupping the Lower Deck
  specular(255, 240, 200);
  shininess(22.0f);
  meshAllClosedPetals.draw();

  // 6E. 32 Fast-Rotating Lower Lotus Blades & Motion-Blur Halo Below Lower Deck (y = -5.35m)
  drawLowerRotorAndBlurHalo();

  popMatrix();

  // 7. Drifting Sunlit Cumulus Cloud Banks (after Vimana for depth occlusion + 1 light pass)
  drawClouds(flightSpeed, dt);

  // 8. Render 2D Glassmorphic / Golden Touch HUD Overlay
  drawHUD();
}

// ============================================================================
// 3. GOLDEN-HOUR LIGHTING SETUP (STRICTLY <= 8 HARDWARE/SHADER LIGHTS)
// ============================================================================
void setupSceneLights(float hoverY) {
  noLights();
  ambientLight(158, 134, 102); // Light #1

  lightSpecular(165, 136, 86);
  directionalLight(255, 214, 146, sunDirLight.x, sunDirLight.y, sunDirLight.z);       // Light #2
  lightSpecular(0, 0, 0);
  directionalLight(235, 172, 102, bounceDirLight.x, bounceDirLight.y, bounceDirLight.z); // Light #3

  lightFalloff(1.0f, 0.055f, 0.005f);
  // Upper deck warm glow (Lights #4 & #5)
  pointLight(255, 156, 64,  5.8f, -(2.8f + hoverY),  2.8f);
  pointLight(255,  96, 54, -4.8f, -(2.7f + hoverY),  4.2f);
  // Lower deck warm glow illuminating the Woman & lower colonnade (Lights #6 & #7)
  pointLight(255, 178, 76,  4.9f, -(-1.8f + hoverY),  2.9f);
  pointLight(255, 110, 58, -4.4f, -(-1.9f + hoverY), -3.6f);
  lightFalloff(1.0f, 0.0f, 0.0f);
}

// ============================================================================
// 4. CHARACTERS & ROTOR RENDERING (RETAINED VBOs + ZERO-ALLOC ANIMATIONS)
// ============================================================================
void drawWoman() {
  pushMatrix();
  translate(womanX, -LOWER_DECK_Y, womanZ);
  rotateY(-WOMAN_ANGLE - PI * 0.58f);

  noStroke();
  specular(60, 48, 36);
  shininess(8.0f);

  meshWomanBodyStatic.draw();

  // Upper Body Group (breathing animation at y = 0.96m)
  pushMatrix();
  translate(0.0f, -0.96f, 0.0f);
  float breathScale = 1.0f + sin(elapsedTime * 2.1f) * 0.008f;
  scale(1.0f, breathScale, 1.0f);

  // Breeze-Animated Flowing Saree Pallu Tail behind left shoulder
  drawAnimatedPalluTail();

  // Sculpted Head, Facial Features, Red Sindoor/Bindi & Braided Bun (1.68m height)
  pushMatrix();
  translate(0.0f, -0.56f, 0.015f);
  rotateY(sin(elapsedTime * 0.65f) * 0.14f);
  rotateX(-0.08f + cos(elapsedTime * 0.85f) * 0.035f);
  meshWomanHeadFull.draw();
  popMatrix();

  popMatrix(); // end upper body
  popMatrix(); // end woman
}

void drawAnimatedPalluTail() {
  pushMatrix();
  translate(-0.14f, 0.02f, -0.17f);
  rotateZ(0.12f);
  rotateY(0.35f);
  rotateX(0.18f);

  final int rows = 10;
  final float hw = 0.17f;
  final float h = 0.82f;
  final float invRows = 1.0f / rows;
  final float tZ = elapsedTime * 4.6f;
  final float tX = elapsedTime * 3.8f;

  noStroke();
  noTint();
  fill(217, 138, 60);
  beginShape(TRIANGLE_STRIP);
  for (int r = 0; r <= rows; r++) {
    float v = r * invRows;
    float ny = -(-0.41f + v * h);
    float wz = sin(tZ - v * 5.2f) * 0.032f * v;
    float wx = cos(tX - v * 4.0f) * 0.018f * v;
    normal(0.0f, 0.0f, -1.0f);
    vertex(-hw + wx, ny, wz);
    vertex( hw + wx, ny, wz);
  }
  endShape();

  popMatrix();
}

void drawMan() {
  pushMatrix();
  translate(manX, -UPPER_DECK_Y, manZ);
  rotateY(-0.55f);

  noStroke();
  specular(110, 88, 54);
  shininess(18.0f);
  meshManBodyStatic.draw();

  // Upper Torso Group (breathing & commanding posture at y = 0.98m)
  pushMatrix();
  translate(0.0f, -0.98f, 0.0f);
  float breathScale = 1.0f + sin(elapsedTime * 1.85f) * 0.009f;
  scale(1.0f, breathScale, 1.0f);

  // Embossed Charcoal-Black & Gold Cuirass (Kavacha)
  specular(210, 172, 96);
  shininess(30.0f);
  pushMatrix();
  scale(1.26f, 1.0f, 0.84f);
  shape(shapeManArmorTorso);
  popMatrix();

  // Sculpted Warrior Head, Tripundra Tilak, Beard & Flowing Hair (1.84m)
  specular(72, 54, 40);
  shininess(10.0f);
  pushMatrix();
  translate(0.0f, -0.68f, 0.022f);
  rotateY(0.14f + sin(elapsedTime * 0.55f) * 0.08f);
  rotateX(0.14f);
  meshManHeadFull.draw();
  popMatrix();

  popMatrix(); // end upper torso
  popMatrix(); // end man
}

void drawLowerRotorAndBlurHalo() {
  specular(255, 224, 145);
  shininess(34.0f);

  pushMatrix();
  translate(0.0f, 5.35f, 0.0f); // y = -5.35m in world space

  // Spinning Rotor Hub & 32 Lotus Blades
  pushMatrix();
  rotateY(rotorAngle);
  shape(shapeRotorHub);
  meshAllRotorBlades.draw();
  popMatrix();

  // Speed-Reactive Golden Motion-Blur Halo Disc (immediate textured quad)
  float blurFactor = constrain((rotorSpeed - 2.5f) / 14.0f, 0.0f, 1.0f);
  if (blurFactor > 0.02f) {
    hint(DISABLE_DEPTH_MASK);
    blendMode(ADD);
    noLights();
    noStroke();

    pushMatrix();
    rotateY(rotorAngle * 1.75f);
    float r = 22.6f;
    float a = blurFactor * 145.0f;
    textureMode(NORMAL);
    beginShape(TRIANGLES);
    texture(texBlurHalo);
    tint(255, 205, 102, a);
    fill(255, 205, 102, a);
    vertex(-r, 0.0f, -r, 0, 0);
    vertex( r, 0.0f, -r, 1, 0);
    vertex( r, 0.0f,  r, 1, 1);

    vertex(-r, 0.0f, -r, 0, 0);
    vertex( r, 0.0f,  r, 1, 1);
    vertex(-r, 0.0f,  r, 0, 1);
    endShape();
    noTint();
    popMatrix();

    blendMode(BLEND);
    hint(ENABLE_DEPTH_MASK);
  }

  popMatrix();
}

// ============================================================================
// 5. SUNLIT CUMULUS CLOUDS (SINGLE IMMEDIATE STREAMING BATCH; ZERO SETVERTEX)
// ============================================================================
class CloudPuff {
  float x, y, z, hw, hh, alpha;
  int col;
  CloudPuff(float x, float y, float z, float w, float h, int col, float alpha) {
    this.x = x; this.y = y; this.z = z;
    this.hw = w * 0.5f; this.hh = h * 0.5f;
    this.col = col; this.alpha = alpha;
  }
}

void initClouds() {
  float spread = 460.0f;
  for (int i = 0; i < CLOUD_COUNT; i++) {
    float cx = random(-spread, spread);
    float cy = -36.0f - random(44.0f);
    float cz = random(-spread, spread);
    float sc = 65.0f + random(95.0f);
    int col = (i % 3 == 0) ? color(255, 230, 189) : color(255, 243, 222);
    float alpha = 175.0f + random(55.0f);
    clouds[i] = new CloudPuff(cx, cy, cz, sc * 1.45f, sc * 0.85f, col, alpha);
  }
}

void drawClouds(float flightSpeed, float dt) {
  noLights();
  noStroke();
  hint(DISABLE_DEPTH_MASK);

  scratchCamFwd.set(camTarget.x - camPos.x, 0.0f, camTarget.z - camPos.z);
  if (scratchCamFwd.magSq() < 0.0001f) scratchCamFwd.set(0, 0, -1);
  scratchCamFwd.normalize();
  scratchCamRight.set(-scratchCamFwd.z, 0.0f, scratchCamFwd.x);

  float rx = scratchCamRight.x;
  float rz = scratchCamRight.z;
  float dzStep = flightSpeed * dt;

  textureMode(NORMAL);
  beginShape(TRIANGLES);
  texture(texCloud);

  for (int i = 0; i < CLOUD_COUNT; i++) {
    CloudPuff cp = clouds[i];
    cp.z += dzStep;
    if (cp.z > 460.0f) cp.z -= 920.0f;

    float hw = cp.hw;
    float hh = cp.hh;
    float py0 = -cp.y - hh;
    float py1 = -cp.y + hh;
    float dx = rx * hw;
    float dz = rz * hw;

    float xL = cp.x - dx, zL = cp.z - dz;
    float xR = cp.x + dx, zR = cp.z + dz;

    tint(cp.col, cp.alpha);
    fill(cp.col, cp.alpha);
    vertex(xL, py0, zL, 0, 0);
    vertex(xR, py0, zR, 1, 0);
    vertex(xR, py1, zR, 1, 1);

    vertex(xL, py0, zL, 0, 0);
    vertex(xR, py1, zR, 1, 1);
    vertex(xL, py1, zL, 0, 1);
  }

  endShape();
  noTint();
  hint(ENABLE_DEPTH_MASK);
}

// ============================================================================
// 6. SAFE AUTO-CHUNKING VBO MESH BUILDER (<= 15,000 VERTICES PER GLES BUFFER)
// ============================================================================
class MultiMesh {
  ArrayList<PShape> chunkList = new ArrayList<PShape>();
  PShape[] chunks = new PShape[0];

  void bakeArray() {
    chunks = chunkList.toArray(new PShape[chunkList.size()]);
    chunkList.clear();
  }

  void draw() {
    for (int i = 0; i < chunks.length; i++) {
      shape(chunks[i]);
    }
  }
}

class MeshBuilder {
  static final int MAX_VERTS_PER_CHUNK = 15000;
  static final int MAX_MATRIX_DEPTH = 32;
  MultiMesh out = new MultiMesh();
  PShape curr;
  int vertCount = 0;
  PImage tex;
  int tintCol;

  PMatrix3D mat = new PMatrix3D();
  PMatrix3D cachedNMat = new PMatrix3D();
  boolean normalMatDirty = true;
  PMatrix3D[] stack = new PMatrix3D[MAX_MATRIX_DEPTH];
  int stackDepth = 0;
  PVector tmpP = new PVector();
  PVector tmpN = new PVector();

  HashMap<Integer, float[]> cosCache = new HashMap<Integer, float[]>();
  HashMap<Integer, float[]> sinCache = new HashMap<Integer, float[]>();

  MeshBuilder(PImage tex, int tintCol) {
    this.tex = tex;
    this.tintCol = tintCol;
    for (int i = 0; i < MAX_MATRIX_DEPTH; i++) {
      stack[i] = new PMatrix3D();
    }
    startChunk();
  }

  float[] getCosTable(int segs) {
    float[] tbl = cosCache.get(segs);
    if (tbl == null) {
      tbl = new float[segs + 1];
      float inv = TWO_PI / segs;
      for (int i = 0; i <= segs; i++) tbl[i] = cos(i * inv);
      cosCache.put(segs, tbl);
    }
    return tbl;
  }

  float[] getSinTable(int segs) {
    float[] tbl = sinCache.get(segs);
    if (tbl == null) {
      tbl = new float[segs + 1];
      float inv = TWO_PI / segs;
      for (int i = 0; i <= segs; i++) tbl[i] = sin(i * inv);
      sinCache.put(segs, tbl);
    }
    return tbl;
  }

  void startChunk() {
    curr = createShape();
    curr.beginShape(TRIANGLES);
    curr.noStroke();
    if (tex != null) {
      curr.textureMode(NORMAL);
      curr.texture(tex);
      curr.fill(tintCol);
    }
    vertCount = 0;
  }

  void ensureSpace(int neededVerts) {
    if (vertCount + neededVerts > MAX_VERTS_PER_CHUNK && vertCount > 0) {
      curr.endShape();
      out.chunkList.add(curr);
      startChunk();
    }
  }

  void pushMat() {
    if (stackDepth < MAX_MATRIX_DEPTH) {
      stack[stackDepth++].set(mat);
    }
  }

  void popMat() {
    if (stackDepth > 0) {
      mat.set(stack[--stackDepth]);
      normalMatDirty = true;
    }
  }

  void translate(float x, float y, float z) {
    mat.translate(x, y, z);
    normalMatDirty = true;
  }

  void rotateX(float a) {
    mat.rotateX(a);
    normalMatDirty = true;
  }

  void rotateY(float a) {
    mat.rotateY(a);
    normalMatDirty = true;
  }

  void rotateZ(float a) {
    mat.rotateZ(a);
    normalMatDirty = true;
  }

  void scale(float sx, float sy, float sz) {
    mat.scale(sx, sy, sz);
    normalMatDirty = true;
  }

  PMatrix3D getNormalMatrix() {
    if (normalMatDirty) {
      cachedNMat.set(mat);
      cachedNMat.m03 = 0.0f;
      cachedNMat.m13 = 0.0f;
      cachedNMat.m23 = 0.0f;
      if (!cachedNMat.invert()) {
        cachedNMat.reset();
      } else {
        cachedNMat.transpose();
      }
      normalMatDirty = false;
    }
    return cachedNMat;
  }

  void emitV(PMatrix3D nMat, float x, float y, float z, float nx, float ny, float nz, int col) {
    tmpP.set(x, y, z);
    mat.mult(tmpP, tmpP);
    tmpN.set(nx, ny, nz);
    nMat.mult(tmpN, tmpN);
    if (tmpN.magSq() > 1e-8f) tmpN.normalize();
    else tmpN.set(0.0f, 1.0f, 0.0f);
    curr.normal(tmpN.x, tmpN.y, tmpN.z);
    curr.fill(col);
    curr.vertex(tmpP.x, tmpP.y, tmpP.z);
    vertCount++;
  }

  void emitVTex(PMatrix3D nMat, float x, float y, float z, float nx, float ny, float nz, float u, float v) {
    tmpP.set(x, y, z);
    mat.mult(tmpP, tmpP);
    tmpN.set(nx, ny, nz);
    nMat.mult(tmpN, tmpN);
    if (tmpN.magSq() > 1e-8f) tmpN.normalize();
    else tmpN.set(0.0f, 1.0f, 0.0f);
    curr.normal(tmpN.x, tmpN.y, tmpN.z);
    if (tex != null) {
      curr.vertex(tmpP.x, tmpP.y, tmpP.z, u, v);
    } else {
      curr.vertex(tmpP.x, tmpP.y, tmpP.z);
    }
    vertCount++;
  }

  void addBox(float w, float h, float d, int col) {
    ensureSpace(36);
    float hx = w * 0.5f, hy = h * 0.5f, hz = d * 0.5f;
    PMatrix3D nm = getNormalMatrix();
    addQuad(nm, -hx,-hy, hz,  hx,-hy, hz,  hx, hy, hz, -hx, hy, hz,  0, 0, 1, col);
    addQuad(nm,  hx,-hy,-hz, -hx,-hy,-hz, -hx, hy,-hz,  hx, hy,-hz,  0, 0,-1, col);
    addQuad(nm,  hx,-hy, hz,  hx,-hy,-hz,  hx, hy,-hz,  hx, hy, hz,  1, 0, 0, col);
    addQuad(nm, -hx,-hy,-hz, -hx,-hy, hz, -hx, hy, hz, -hx, hy,-hz, -1, 0, 0, col);
    addQuad(nm, -hx, hy, hz,  hx, hy, hz,  hx, hy,-hz, -hx, hy,-hz,  0, 1, 0, col);
    addQuad(nm, -hx,-hy,-hz,  hx,-hy,-hz,  hx,-hy, hz, -hx,-hy, hz,  0,-1, 0, col);
  }

  void addQuad(PMatrix3D nm, float x0, float y0, float z0, float x1, float y1, float z1,
               float x2, float y2, float z2, float x3, float y3, float z3,
               float nx, float ny, float nz, int col) {
    emitV(nm, x0, y0, z0, nx, ny, nz, col);
    emitV(nm, x1, y1, z1, nx, ny, nz, col);
    emitV(nm, x2, y2, z2, nx, ny, nz, col);
    emitV(nm, x0, y0, z0, nx, ny, nz, col);
    emitV(nm, x2, y2, z2, nx, ny, nz, col);
    emitV(nm, x3, y3, z3, nx, ny, nz, col);
  }

  void addLathe(PVector[] profile, int segs, int col) {
    ensureSpace((profile.length - 1) * segs * 6);
    PMatrix3D nm = getNormalMatrix();
    float[] cosT = getCosTable(segs);
    float[] sinT = getSinTable(segs);

    for (int p = 0; p < profile.length - 1; p++) {
      float r0 = profile[p].x, y0 = profile[p].y;
      float r1 = profile[p + 1].x, y1 = profile[p + 1].y;
      float dr = r1 - r0, dy = y1 - y0;
      float nr = dy, ny = -dr;
      float nLen = sqrt(nr * nr + ny * ny);
      if (nLen > 1e-5f) { nr /= nLen; ny /= nLen; }
      else { nr = 1.0f; ny = 0.0f; }

      for (int i = 0; i < segs; i++) {
        float c0 = cosT[i], s0 = sinT[i];
        float c1 = cosT[i + 1], s1 = sinT[i + 1];

        emitV(nm, c0 * r0, -y0, s0 * r0, c0 * nr, -ny, s0 * nr, col);
        emitV(nm, c1 * r0, -y0, s1 * r0, c1 * nr, -ny, s1 * nr, col);
        emitV(nm, c1 * r1, -y1, s1 * r1, c1 * nr, -ny, s1 * nr, col);

        emitV(nm, c0 * r0, -y0, s0 * r0, c0 * nr, -ny, s0 * nr, col);
        emitV(nm, c1 * r1, -y1, s1 * r1, c1 * nr, -ny, s1 * nr, col);
        emitV(nm, c0 * r1, -y1, s0 * r1, c0 * nr, -ny, s0 * nr, col);
      }
    }
  }

  void addTorusRing(float majorR, float minorR, float worldY, int segs, int tubeSegs, int col) {
    ensureSpace(segs * tubeSegs * 6);
    PMatrix3D nm = getNormalMatrix();
    float[] cosA = getCosTable(segs);
    float[] sinA = getSinTable(segs);
    float[] cosB = getCosTable(tubeSegs);
    float[] sinB = getSinTable(tubeSegs);

    for (int i = 0; i < segs; i++) {
      float ca0 = cosA[i], sa0 = sinA[i];
      float ca1 = cosA[i + 1], sa1 = sinA[i + 1];
      for (int j = 0; j < tubeSegs; j++) {
        float cb0 = cosB[j], sb0 = sinB[j];
        float cb1 = cosB[j + 1], sb1 = sinB[j + 1];

        float r00 = majorR + minorR * cb0;
        float r01 = majorR + minorR * cb1;
        float y0 = -(worldY + minorR * sb0);
        float y1 = -(worldY + minorR * sb1);

        emitV(nm, ca0 * r00, y0, sa0 * r00, ca0 * cb0, -sb0, sa0 * cb0, col);
        emitV(nm, ca1 * r00, y0, sa1 * r00, ca1 * cb0, -sb0, sa1 * cb0, col);
        emitV(nm, ca1 * r01, y1, sa1 * r01, ca1 * cb1, -sb1, sa1 * cb1, col);

        emitV(nm, ca0 * r00, y0, sa0 * r00, ca0 * cb0, -sb0, sa0 * cb0, col);
        emitV(nm, ca1 * r01, y1, sa1 * r01, ca1 * cb1, -sb1, sa1 * cb1, col);
        emitV(nm, ca0 * r01, y1, sa0 * r01, ca0 * cb1, -sb1, sa0 * cb1, col);
      }
    }
  }

  void addVerticalRing(float majorR, float minorR, int segs, int col) {
    ensureSpace(segs * 6);
    PMatrix3D nm = getNormalMatrix();
    float[] cosT = getCosTable(segs);
    float[] sinT = getSinTable(segs);
    float rIn = majorR - minorR;
    float rOut = majorR + minorR;

    for (int i = 0; i < segs; i++) {
      float c0 = cosT[i], s0 = sinT[i];
      float c1 = cosT[i + 1], s1 = sinT[i + 1];

      emitV(nm, c0 * rIn,  s0 * rIn,  0, 0, 0, 1, col);
      emitV(nm, c0 * rOut, s0 * rOut, 0, 0, 0, 1, col);
      emitV(nm, c1 * rOut, s1 * rOut, 0, 0, 0, 1, col);

      emitV(nm, c0 * rIn,  s0 * rIn,  0, 0, 0, 1, col);
      emitV(nm, c1 * rOut, s1 * rOut, 0, 0, 0, 1, col);
      emitV(nm, c1 * rIn,  s1 * rIn,  0, 0, 0, 1, col);
    }
  }

  void addCylinder(float rTop, float rBot, float h, int segs, int col) {
    ensureSpace(segs * 6);
    PMatrix3D nm = getNormalMatrix();
    float[] cosT = getCosTable(segs);
    float[] sinT = getSinTable(segs);
    float hh = h * 0.5f;

    for (int i = 0; i < segs; i++) {
      float c0 = cosT[i], s0 = sinT[i];
      float c1 = cosT[i + 1], s1 = sinT[i + 1];

      emitV(nm, c0 * rTop, -hh, s0 * rTop, c0, 0, s0, col);
      emitV(nm, c1 * rTop, -hh, s1 * rTop, c1, 0, s1, col);
      emitV(nm, c1 * rBot,  hh, s1 * rBot, c1, 0, s1, col);

      emitV(nm, c0 * rTop, -hh, s0 * rTop, c0, 0, s0, col);
      emitV(nm, c1 * rBot,  hh, s1 * rBot, c1, 0, s1, col);
      emitV(nm, c0 * rBot,  hh, s0 * rBot, c0, 0, s0, col);
    }
  }

  void addCone(float r, float h, int segs, int col) {
    ensureSpace(segs * 3);
    PMatrix3D nm = getNormalMatrix();
    float[] cosT = getCosTable(segs);
    float[] sinT = getSinTable(segs);
    float hh = h * 0.5f;

    for (int i = 0; i < segs; i++) {
      float c0 = cosT[i], s0 = sinT[i];
      float c1 = cosT[i + 1], s1 = sinT[i + 1];

      emitV(nm, 0.0f,   -hh, 0.0f,   c0, -0.35f, s0, col);
      emitV(nm, c0 * r,  hh, s0 * r, c0, -0.35f, s0, col);
      emitV(nm, c1 * r,  hh, s1 * r, c1, -0.35f, s1, col);
    }
  }

  void addEllipsoid(float r, float sx, float sy, float sz, int segU, int segV, int col, float maxTheta) {
    ensureSpace(segU * segV * 6);
    PMatrix3D nm = getNormalMatrix();
    float[] cosP = getCosTable(segU);
    float[] sinP = getSinTable(segU);

    for (int iv = 0; iv < segV; iv++) {
      float t0 = (iv / (float) segV) * maxTheta;
      float t1 = ((iv + 1) / (float) segV) * maxTheta;
      float st0 = sin(t0), ct0 = cos(t0);
      float st1 = sin(t1), ct1 = cos(t1);
      for (int iu = 0; iu < segU; iu++) {
        float cp0 = cosP[iu], sp0 = sinP[iu];
        float cp1 = cosP[iu + 1], sp1 = sinP[iu + 1];

        float ax = r * st0 * cp0 * sx, ay = -r * ct0 * sy, az = r * st0 * sp0 * sz;
        float bx = r * st0 * cp1 * sx, by = -r * ct0 * sy, bz = r * st0 * sp1 * sz;
        float cx = r * st1 * cp0 * sx, cy = -r * ct1 * sy, cz = r * st1 * sp0 * sz;
        float dx = r * st1 * cp1 * sx, dy = -r * ct1 * sy, dz = r * st1 * sp1 * sz;

        emitV(nm, ax, ay, az, ax, ay, az, col);
        emitV(nm, bx, by, bz, bx, by, bz, col);
        emitV(nm, dx, dy, dz, dx, dy, dz, col);

        emitV(nm, ax, ay, az, ax, ay, az, col);
        emitV(nm, dx, dy, dz, dx, dy, dz, col);
        emitV(nm, cx, cy, cz, cx, cy, cz, col);
      }
    }
  }

  void addCrownLeaf(float h, float w, float curveOut, int col) {
    ensureSpace(9);
    PMatrix3D nm = getNormalMatrix();
    float hw = w * 0.5f;
    float y2 = -h * 0.55f;
    float y3 = -h;
    addTriRaw(nm, -hw, 0.0f, 0.0f,  hw, 0.0f, 0.0f,  0.0f, y2, curveOut, col);
    addTriRaw(nm,  hw, 0.0f, 0.0f,  0.0f, y3, -0.35f,  0.0f, y2, curveOut, col);
    addTriRaw(nm,  0.0f, y3, -0.35f, -hw, 0.0f, 0.0f,  0.0f, y2, curveOut, col);
  }

  void addTriRaw(PMatrix3D nm, float ax, float ay, float az,
                 float bx, float by, float bz,
                 float cx, float cy, float cz, int col) {
    float abx = bx - ax, aby = by - ay, abz = bz - az;
    float acx = cx - ax, acy = cy - ay, acz = cz - az;
    float nx = aby * acz - abz * acy;
    float ny = abz * acx - abx * acz;
    float nz = abx * acy - aby * acx;
    float len = sqrt(nx * nx + ny * ny + nz * nz);
    if (len > 1e-8f) { nx /= len; ny /= len; nz /= len; }
    else { nx = 0.0f; ny = 1.0f; nz = 0.0f; }
    emitV(nm, ax, ay, az, nx, ny, nz, col);
    emitV(nm, bx, by, bz, nx, ny, nz, col);
    emitV(nm, cx, cy, cz, nx, ny, nz, col);
  }

  MultiMesh finish() {
    if (vertCount > 0) {
      curr.endShape();
      out.chunkList.add(curr);
    }
    out.bakeArray();
    return out;
  }
}

// ============================================================================
// 7. 3D MESH & BAKED ASSEMBLY BUILDERS
// ============================================================================
void buildDeckSlabsAndDomeShapes() {
  int deckGoldTint = color(255, 223, 153);

  // 1. Merged Upper Deck + Lower Deck + Ceiling Disc (3,672 verts)
  PVector[] upperDeckProfile = {
    new PVector(4.20f,  0.00f),
    new PVector(9.50f,  0.00f),
    new PVector(9.65f, -0.14f),
    new PVector(9.45f, -0.34f),
    new PVector(9.10f, -0.62f),
    new PVector(4.35f, -0.62f),
    new PVector(4.15f, -0.32f),
    new PVector(4.20f,  0.00f)
  };
  PVector[] lowerDeckProfile = {
    new PVector(3.60f, -4.20f),
    new PVector(8.40f, -4.20f),
    new PVector(8.55f, -4.34f),
    new PVector(8.35f, -4.52f),
    new PVector(8.05f, -4.75f),
    new PVector(3.75f, -4.75f),
    new PVector(3.55f, -4.48f),
    new PVector(3.60f, -4.20f)
  };
  PVector[] ceilingProfile = {
    new PVector(0.00f, -0.17f),
    new PVector(9.55f, -0.17f),
    new PVector(9.55f,  0.17f),
    new PVector(0.00f,  0.17f)
  };

  shapeDeckSlabsAndCeiling = createShape();
  shapeDeckSlabsAndCeiling.beginShape(TRIANGLES);
  shapeDeckSlabsAndCeiling.noStroke();
  shapeDeckSlabsAndCeiling.textureMode(NORMAL);
  shapeDeckSlabsAndCeiling.texture(texDeckFloor);
  shapeDeckSlabsAndCeiling.fill(deckGoldTint);
  appendLatheToShape(shapeDeckSlabsAndCeiling, upperDeckProfile, 36, true, true, 0.0f);
  appendLatheToShape(shapeDeckSlabsAndCeiling, lowerDeckProfile, 36, true, true, 0.0f);
  appendLatheToShape(shapeDeckSlabsAndCeiling, ceilingProfile,   36, true, true, -4.60f);
  shapeDeckSlabsAndCeiling.endShape();

  // 2. Merged Central Carved Golden Axis Column + Bulbous Kalasha Dome (5,400 verts)
  PVector[] centralColProfile = {
    new PVector(0.00f, -5.85f),
    new PVector(1.38f, -5.65f),
    new PVector(1.12f, -4.75f),
    new PVector(1.22f, -4.20f),
    new PVector(1.02f, -3.65f),
    new PVector(0.94f, -2.55f),
    new PVector(1.04f, -2.15f),
    new PVector(0.94f, -1.75f),
    new PVector(1.18f, -0.75f),
    new PVector(1.28f, -0.10f),
    new PVector(1.22f,  0.35f),
    new PVector(1.04f,  0.65f),
    new PVector(0.96f,  1.80f),
    new PVector(1.06f,  2.20f),
    new PVector(0.96f,  2.55f),
    new PVector(0.96f,  3.65f),
    new PVector(1.26f,  4.15f),
    new PVector(1.52f,  4.55f),
    new PVector(0.00f,  4.60f)
  };
  PVector[] domeProfile = {
    new PVector(0.00f,  4.60f),
    new PVector(6.75f,  4.60f),
    new PVector(6.95f,  5.10f),
    new PVector(6.55f,  5.65f),
    new PVector(7.35f,  7.10f),
    new PVector(7.68f,  9.10f),
    new PVector(7.42f, 11.20f),
    new PVector(6.48f, 13.00f),
    new PVector(4.95f, 14.35f),
    new PVector(4.45f, 14.75f),
    new PVector(4.75f, 15.10f),
    new PVector(3.65f, 16.05f),
    new PVector(2.15f, 16.65f),
    new PVector(0.00f, 16.85f)
  };

  shapeColumnAndDome = createShape();
  shapeColumnAndDome.beginShape(TRIANGLES);
  shapeColumnAndDome.noStroke();
  shapeColumnAndDome.textureMode(NORMAL);
  shapeColumnAndDome.texture(texDomeFiligree);
  shapeColumnAndDome.fill(deckGoldTint);
  appendLatheToShape(shapeColumnAndDome, centralColProfile, 24, true, false, 0.0f);
  appendLatheToShape(shapeColumnAndDome, domeProfile,       36, true, false, 0.0f);
  shapeColumnAndDome.endShape();
}

void buildUpperDeckStaticShapes() {
  MeshBuilder mb = new MeshBuilder(null, color(255));

  PVector[] balusterProfile = {
    new PVector(0.065f, 0.08f),
    new PVector(0.038f, 0.34f),
    new PVector(0.072f, 0.56f),
    new PVector(0.036f, 0.82f),
    new PVector(0.062f, 1.02f)
  };

  PVector[] upperPillarProfile = {
    new PVector(0.42f, 0.00f),
    new PVector(0.42f, 0.32f),
    new PVector(0.32f, 0.55f),
    new PVector(0.27f, 2.15f),
    new PVector(0.34f, 2.32f),
    new PVector(0.26f, 2.48f),
    new PVector(0.26f, 3.85f),
    new PVector(0.44f, 4.25f),
    new PVector(0.52f, 4.58f)
  };

  int goldBright = color(244, 190, 82);
  int goldDeep   = color(217, 155, 44);
  int crimsonCol = color(128, 28, 24);
  int bannerGold = color(238, 182, 68);

  for (int i = 0; i < 8; i++) {
    mb.pushMat();
    mb.rotateY((i / 8.0f) * TWO_PI);
    mb.translate(0.0f, 0.46f, 2.68f);
    mb.addBox(0.28f, 0.24f, 3.25f, goldDeep);
    mb.popMat();
  }

  mb.addTorusRing(4.26f, 0.075f, 1.04f, 32, 6, goldBright);
  mb.addTorusRing(4.26f, 0.09f,  0.08f, 32, 6, goldDeep);

  int balusterCount = 32;
  for (int i = 0; i < balusterCount; i++) {
    float ang = (i / (float) balusterCount) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 4.26f, 0.0f, sin(ang) * 4.26f);
    mb.addLathe(balusterProfile, 6, goldBright);
    mb.popMat();

    float midAng = ang + PI / balusterCount;
    mb.pushMat();
    mb.translate(cos(midAng) * 4.26f, -0.56f, sin(midAng) * 4.26f);
    mb.rotateY(-midAng + HALF_PI);
    mb.addVerticalRing(0.14f, 0.018f, 8, goldDeep);
    mb.popMat();
  }

  int pillarCount = 12;
  float spanLen = 2.0f * 8.80f * sin(PI / pillarCount);
  for (int i = 0; i < pillarCount; i++) {
    float ang = (i / (float) pillarCount) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 8.80f, 0.0f, sin(ang) * 8.80f);
    mb.addLathe(upperPillarProfile, 10, goldDeep);
    mb.popMat();

    float midAng = ang + PI / pillarCount;
    mb.pushMat();
    mb.translate(cos(midAng) * 8.80f, -4.35f, sin(midAng) * 8.80f);
    mb.rotateY(-midAng + HALF_PI);
    mb.addBox(spanLen, 0.44f, 0.42f, goldBright);
    mb.popMat();

    if (i % 2 == 0) {
      mb.pushMat();
      mb.translate(cos(ang) * 9.02f, -2.35f, sin(ang) * 9.02f);
      mb.rotateY(-ang + HALF_PI);
      mb.addBox(0.56f, 3.65f, 0.06f, crimsonCol);
      mb.addBox(0.64f, 3.65f, 0.03f, bannerGold);
      mb.popMat();
    }
  }

  mb.addTorusRing(9.25f, 0.08f, 0.92f, 32, 6, goldBright);
  for (int i = 0; i < 36; i++) {
    float ang = (i / 36.0f) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 9.25f, 0.0f, sin(ang) * 9.25f);
    mb.scale(0.9f, 0.88f, 0.9f);
    mb.addLathe(balusterProfile, 6, goldBright);
    mb.popMat();
  }

  // Ceiling radial ribs & corbels
  mb.pushMat();
  mb.translate(0.0f, -4.60f, 0.0f);
  for (int i = 0; i < 24; i++) {
    mb.pushMat();
    mb.rotateY((i / 24.0f) * TWO_PI);

    mb.pushMat();
    mb.translate(0.0f, 0.22f, 5.25f);
    mb.addBox(0.18f, 0.22f, 8.2f, goldBright);
    mb.popMat();

    mb.pushMat();
    mb.translate(0.0f, 0.24f, 8.65f);
    mb.scale(1.0f, 0.28f, 1.0f);
    mb.addBox(0.45f, 0.45f, 0.78f, goldBright);
    mb.popMat();

    mb.popMat();
  }
  mb.popMat();

  meshUpperDeckStatic = mb.finish();
}

void buildLowerDeckAndFinialStaticShapes() {
  MeshBuilder mb = new MeshBuilder(null, color(255));

  PVector[] balusterProfile = {
    new PVector(0.065f, 0.08f),
    new PVector(0.038f, 0.34f),
    new PVector(0.072f, 0.56f),
    new PVector(0.036f, 0.82f),
    new PVector(0.062f, 1.02f)
  };

  PVector[] lowerPillarProfile = {
    new PVector(0.38f, -4.20f),
    new PVector(0.38f, -3.92f),
    new PVector(0.29f, -3.72f),
    new PVector(0.25f, -2.55f),
    new PVector(0.31f, -2.40f),
    new PVector(0.25f, -2.25f),
    new PVector(0.25f, -1.15f),
    new PVector(0.40f, -0.85f),
    new PVector(0.48f, -0.62f)
  };

  int goldBright = color(244, 190, 82);
  int goldDeep   = color(217, 155, 44);
  int crimsonCol = color(128, 28, 24);
  int bannerGold = color(238, 182, 68);

  for (int i = 0; i < 8; i++) {
    mb.pushMat();
    mb.rotateY((i / 8.0f) * TWO_PI);
    mb.translate(0.0f, 4.60f, 2.38f);
    mb.addBox(0.26f, 0.22f, 2.75f, goldDeep);
    mb.popMat();
  }

  mb.addTorusRing(3.66f, 0.07f,  LOWER_DECK_Y + 1.04f, 32, 6, goldBright);
  mb.addTorusRing(3.66f, 0.085f, LOWER_DECK_Y + 0.08f, 32, 6, goldDeep);

  int innerBalusters = 28;
  for (int i = 0; i < innerBalusters; i++) {
    float ang = (i / (float) innerBalusters) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 3.66f, -LOWER_DECK_Y, sin(ang) * 3.66f);
    mb.addLathe(balusterProfile, 6, goldBright);
    mb.popMat();

    float midAng = ang + PI / innerBalusters;
    mb.pushMat();
    mb.translate(cos(midAng) * 3.66f, -(LOWER_DECK_Y + 0.56f), sin(midAng) * 3.66f);
    mb.rotateY(-midAng + HALF_PI);
    mb.addVerticalRing(0.13f, 0.017f, 8, goldDeep);
    mb.popMat();
  }

  int pillarCount = 12;
  float lowerSpanLen = 2.0f * 7.75f * sin(PI / pillarCount);
  for (int i = 0; i < pillarCount; i++) {
    float ang = (i / (float) pillarCount) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 7.75f, 0.0f, sin(ang) * 7.75f);
    mb.addLathe(lowerPillarProfile, 10, goldDeep);
    mb.popMat();

    float midAng = ang + PI / pillarCount;
    mb.pushMat();
    mb.translate(cos(midAng) * 7.75f, 0.82f, sin(midAng) * 7.75f);
    mb.rotateY(-midAng + HALF_PI);
    mb.addBox(lowerSpanLen, 0.38f, 0.38f, goldBright);
    mb.popMat();

    if (i % 2 == 1) {
      mb.pushMat();
      mb.translate(cos(ang) * 7.95f, 2.40f, sin(ang) * 7.95f);
      mb.rotateY(-ang + HALF_PI);
      mb.addBox(0.50f, 2.85f, 0.05f, crimsonCol);
      mb.addBox(0.58f, 2.85f, 0.025f, bannerGold);
      mb.popMat();
    }
  }

  mb.addTorusRing(8.15f, 0.075f, LOWER_DECK_Y + 0.92f, 32, 6, goldBright);
  for (int i = 0; i < 32; i++) {
    float ang = (i / 32.0f) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 8.15f, -LOWER_DECK_Y, sin(ang) * 8.15f);
    mb.scale(0.88f, 0.88f, 0.88f);
    mb.addLathe(balusterProfile, 6, goldBright);
    mb.popMat();
  }

  // Dome cornice rings & crown finial
  mb.addTorusRing(6.62f, 0.14f,  5.65f, 32, 6, goldBright);
  mb.addTorusRing(7.72f, 0.14f,  9.10f, 32, 6, goldBright);
  mb.addTorusRing(4.78f, 0.14f, 14.75f, 32, 6, goldBright);

  mb.pushMat();
  mb.translate(0.0f, -15.6f, 0.0f);
  for (int i = 0; i < 8; i++) {
    float ang = (i / 8.0f) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 3.15f, -0.1f, sin(ang) * 3.15f);
    mb.rotateY(-ang + HALF_PI);
    mb.rotateX(-0.18f);
    mb.addCrownLeaf(2.75f, 1.85f, 0.35f, goldBright);
    mb.popMat();
  }
  for (int i = 0; i < 8; i++) {
    float ang = ((i + 0.5f) / 8.0f) * TWO_PI;
    mb.pushMat();
    mb.translate(cos(ang) * 1.85f, -0.65f, sin(ang) * 1.85f);
    mb.rotateY(-ang + HALF_PI);
    mb.rotateX(-0.36f);
    mb.addCrownLeaf(3.45f, 1.45f, 0.22f, goldBright);
    mb.popMat();
  }
  mb.pushMat();
  mb.translate(0.0f, -2.35f, 0.0f);
  mb.addCone(0.42f, 1.65f, 10, goldBright);
  mb.popMat();
  mb.popMat();

  meshLowerDeckAndFinialStatic = mb.finish();
}

void buildLotusPetalsAndRotorShapes() {
  // 1. 24 Closed Curved Lotus Petals (13,824 verts -> single chunk)
  MeshBuilder mbPetals = new MeshBuilder(texUpperPetal, color(255));
  for (int i = 0; i < 12; i++) {
    float angIn = (i / 12.0f) * TWO_PI;
    mbPetals.pushMat();
    mbPetals.translate(cos(angIn) * 8.55f, 4.75f, sin(angIn) * 8.55f);
    mbPetals.rotateY(-angIn + HALF_PI);
    appendClosedCurvedLotusPetal(mbPetals, 6.60f, 4.45f, 3.15f, 1.05f, 0.30f, 0.65f, 8, 12);
    mbPetals.popMat();

    float angOut = ((i + 0.5f) / 12.0f) * TWO_PI;
    mbPetals.pushMat();
    mbPetals.translate(cos(angOut) * 8.85f, 4.90f, sin(angOut) * 8.85f);
    mbPetals.rotateY(-angOut + HALF_PI);
    appendClosedCurvedLotusPetal(mbPetals, 5.75f, 4.95f, 3.55f, 1.15f, 0.38f, 0.72f, 8, 12);
    mbPetals.popMat();
  }
  meshAllClosedPetals = mbPetals.finish();

  // 2. Central Golden Rotor Hub Drum (720 verts)
  PVector[] hubProfile = {
    new PVector(0.00f, -1.15f),
    new PVector(1.15f, -0.95f),
    new PVector(2.35f, -0.45f),
    new PVector(2.65f,  0.00f),
    new PVector(2.35f,  0.45f),
    new PVector(1.45f,  0.65f),
    new PVector(0.00f,  0.65f)
  };
  shapeRotorHub = buildLatheShape(hubProfile, 20, null, color(217, 155, 44), false);

  // 3. 32 Lower Rotor Lotus Blades (11,520 verts -> single chunk)
  MeshBuilder mbRotor = new MeshBuilder(texLowerRotor, color(255, 215, 130));
  for (int i = 0; i < 16; i++) {
    mbRotor.pushMat();
    mbRotor.translate(0.0f, -0.18f, 0.0f);
    mbRotor.rotateY((i / 16.0f) * TWO_PI);
    appendLowerRotorBlade(mbRotor, 20.0f, 4.35f, 0.24f, 6, 10);
    mbRotor.popMat();

    mbRotor.pushMat();
    mbRotor.translate(0.0f, 0.26f, 0.0f);
    mbRotor.rotateY(((i + 0.5f) / 16.0f) * TWO_PI);
    appendLowerRotorBlade(mbRotor, 18.6f, 4.15f, 0.22f, 6, 10);
    mbRotor.popMat();
  }
  meshAllRotorBlades = mbRotor.finish();
}

void buildWomanShapes() {
  int skinCol = color(217, 155, 114);
  int sareeCol = color(217, 138, 60);
  int blouseCol = color(184, 74, 24);
  int goldCol = color(244, 190, 82);
  int hairCol = color(23, 18, 16);

  MeshBuilder mb = new MeshBuilder(null, color(255));

  for (int side = -1; side <= 1; side += 2) {
    mb.pushMat();
    mb.translate(side * 0.095f, -0.016f, 0.02f);
    mb.addBox(0.082f, 0.032f, 0.21f, skinCol);
    mb.popMat();
  }

  PVector[] skirtProfile = {
    new PVector(0.000f, 0.03f),
    new PVector(0.210f, 0.03f),
    new PVector(0.195f, 0.35f),
    new PVector(0.175f, 0.68f),
    new PVector(0.155f, 0.96f),
    new PVector(0.000f, 0.96f)
  };
  mb.pushMat();
  mb.scale(1.08f, 1.0f, 0.86f);
  mb.addLathe(skirtProfile, 16, sareeCol);
  mb.addTorusRing(0.205f, 0.012f, 0.065f, 16, 5, goldCol);
  mb.popMat();

  for (int p = -2; p <= 2; p++) {
    mb.pushMat();
    mb.translate(p * 0.024f, -0.49f, 0.155f);
    mb.rotateX(-0.045f);
    mb.addBox(0.026f, 0.91f, 0.026f, sareeCol);
    mb.popMat();
  }

  mb.pushMat();
  mb.translate(0.0f, -0.96f, 0.0f);

  PVector[] torsoProfile = {
    new PVector(0.000f, 0.00f),
    new PVector(0.152f, 0.00f),
    new PVector(0.142f, 0.14f),
    new PVector(0.172f, 0.27f),
    new PVector(0.182f, 0.36f),
    new PVector(0.085f, 0.40f),
    new PVector(0.000f, 0.40f)
  };
  mb.pushMat();
  mb.scale(1.12f, 1.0f, 0.82f);
  mb.addLathe(torsoProfile, 14, blouseCol);
  mb.popMat();

  PVector[] palluProfile = {
    new PVector(0.175f, 0.00f),
    new PVector(0.165f, 0.42f)
  };
  mb.pushMat();
  mb.rotateZ(-0.34f);
  mb.scale(1.15f, 1.0f, 0.88f);
  mb.addLathe(palluProfile, 12, sareeCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.21f, -0.24f, 0.08f);
  mb.rotateZ(-0.22f);
  mb.rotateX(-0.55f);
  mb.addCylinder(0.044f, 0.038f, 0.28f, 8, skinCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.25f, -0.11f, 0.25f);
  mb.rotateZ(-0.10f);
  mb.rotateX(-1.12f);
  mb.addCylinder(0.036f, 0.028f, 0.27f, 8, skinCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.26f, -0.07f, 0.38f);
  mb.scale(0.85f, 0.65f, 1.35f);
  mb.addEllipsoid(0.034f, 1, 1, 1, 6, 4, skinCol, PI);
  mb.popMat();

  mb.pushMat();
  mb.translate(-0.20f, -0.22f, 0.02f);
  mb.rotateZ(0.18f);
  mb.rotateX(-0.15f);
  mb.addCylinder(0.044f, 0.038f, 0.28f, 8, skinCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(-0.21f, 0.01f, 0.09f);
  mb.rotateZ(-0.06f);
  mb.rotateX(-0.42f);
  mb.addCylinder(0.036f, 0.028f, 0.26f, 8, skinCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.255f, -0.085f, 0.33f);
  mb.rotateX(0.35f);
  mb.addTorusRing(0.033f, 0.006f, 0.0f, 10, 5, goldCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(-0.205f, 0.10f, 0.13f);
  mb.rotateX(0.35f);
  mb.addTorusRing(0.033f, 0.006f, 0.0f, 10, 5, goldCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.0f, -0.42f, 0.01f);
  mb.addCylinder(0.048f, 0.056f, 0.10f, 8, skinCol);
  mb.popMat();

  int beadCol = color(43, 24, 16);
  for (int b = 0; b <= 10; b++) {
    float t = b / 10.0f;
    float ang = lerp(-1.15f, 1.15f, t);
    float nx = sin(ang) * 0.072f;
    float nz = cos(ang) * 0.145f - 0.01f;
    float ny = 0.21f + pow(abs(t - 0.5f) * 2.0f, 1.6f) * 0.18f;
    mb.pushMat();
    mb.translate(nx, -ny, nz);
    mb.addEllipsoid(0.0082f, 1, 1, 1, 6, 4, beadCol, PI);
    mb.popMat();
  }

  mb.popMat();
  meshWomanBodyStatic = mb.finish();

  // Sculpted Woman Head & Facial Features
  MeshBuilder mh = new MeshBuilder(null, color(255));
  mh.addEllipsoid(0.102f, 0.86f, 1.08f, 0.95f, 12, 9, skinCol, PI);

  mh.pushMat();
  mh.translate(0.0f, 0.008f, 0.098f);
  mh.addBox(0.016f, 0.026f, 0.022f, skinCol);
  mh.popMat();

  mh.pushMat();
  mh.translate(-0.011f, 0.014f, 0.104f);
  mh.addEllipsoid(0.004f, 1, 1, 1, 6, 4, goldCol, PI);
  mh.popMat();

  for (int side = -1; side <= 1; side += 2) {
    mh.pushMat();
    mh.translate(side * 0.033f, -0.014f, 0.092f);
    mh.scale(1.2f, 0.75f, 0.6f);
    mh.addEllipsoid(0.010f, 1, 1, 1, 6, 4, color(26, 18, 14), PI);
    mh.popMat();

    mh.pushMat();
    mh.translate(side * 0.034f, -0.028f, 0.094f);
    mh.rotateZ(-side * 0.1f);
    mh.addBox(0.028f, 0.004f, 0.008f, hairCol);
    mh.popMat();
  }

  mh.pushMat();
  mh.translate(0.0f, 0.042f, 0.093f);
  mh.scale(1.35f, 0.45f, 0.65f);
  mh.addEllipsoid(0.016f, 1, 1, 1, 6, 4, color(166, 75, 66), PI);
  mh.popMat();

  int bindiCol = color(217, 22, 22);
  mh.pushMat();
  mh.translate(0.0f, -0.038f, 0.098f);
  mh.addEllipsoid(0.0055f, 1, 1, 1, 6, 4, bindiCol, PI);
  mh.popMat();

  mh.pushMat();
  mh.translate(0.0f, -0.096f, 0.052f);
  mh.rotateX(0.52f);
  mh.addBox(0.008f, 0.012f, 0.055f, bindiCol);
  mh.popMat();

  mh.pushMat();
  mh.translate(0.0f, -0.012f, -0.012f);
  mh.rotateX(-0.24f);
  mh.addEllipsoid(0.108f, 0.90f, 1.08f, 0.99f, 12, 8, hairCol, PI * 0.68f);
  mh.popMat();

  mh.pushMat();
  mh.translate(0.0f, 0.025f, -0.105f);
  mh.addEllipsoid(0.056f, 1, 1, 1, 8, 6, hairCol, PI);
  mh.popMat();

  meshWomanHeadFull = mh.finish();
}

void buildManShapes() {
  int skinCol = color(198, 134, 88);
  int dhotiCol = color(92, 24, 24);
  int goldCol = color(242, 188, 76);
  int goldRich = color(244, 190, 82);
  int sandalCol = color(52, 34, 22);
  int hairCol = color(20, 16, 15);

  MeshBuilder mb = new MeshBuilder(null, color(255));

  for (int side = -1; side <= 1; side += 2) {
    float footX = side * 0.175f;
    float footYaw = -side * 0.14f;

    mb.pushMat();
    mb.translate(footX, 0.0f, 0.0f);
    mb.rotateY(footYaw);

    mb.pushMat();
    mb.translate(0.0f, -0.018f, 0.035f);
    mb.addBox(0.105f, 0.036f, 0.26f, sandalCol);
    mb.popMat();

    mb.pushMat();
    mb.translate(0.0f, -0.042f, 0.045f);
    mb.addBox(0.112f, 0.016f, 0.065f, goldCol);
    mb.popMat();

    mb.pushMat();
    mb.translate(0.0f, -0.28f, 0.0f);
    mb.addCylinder(0.068f, 0.052f, 0.48f, 8, skinCol);
    mb.popMat();

    mb.addTorusRing(0.056f, 0.008f, 0.075f, 12, 5, goldRich);
    mb.popMat();
  }

  PVector[] manDhotiProfile = {
    new PVector(0.000f, 0.36f),
    new PVector(0.248f, 0.36f),
    new PVector(0.236f, 0.58f),
    new PVector(0.222f, 0.80f),
    new PVector(0.205f, 0.98f),
    new PVector(0.000f, 0.98f)
  };
  mb.pushMat();
  mb.scale(1.18f, 1.0f, 0.88f);
  mb.addLathe(manDhotiProfile, 16, dhotiCol);
  mb.addTorusRing(0.245f, 0.014f, 0.40f, 16, 5, goldRich);
  mb.addTorusRing(0.205f, 0.022f, 0.96f, 16, 5, goldRich);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.0f, -0.66f, 0.185f);
  mb.rotateX(-0.04f);
  mb.addBox(0.11f, 0.58f, 0.038f, color(96, 26, 24));
  mb.popMat();

  mb.pushMat();
  mb.translate(0.0f, -0.66f, 0.192f);
  mb.rotateX(-0.04f);
  mb.addBox(0.028f, 0.58f, 0.042f, goldRich);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.0f, -0.98f, 0.0f);

  mb.pushMat();
  mb.translate(0.0f, -0.49f, 0.02f);
  mb.scale(1.18f, 1.0f, 0.88f);
  mb.addTorusRing(0.115f, 0.015f, 0.0f, 14, 5, goldRich);
  mb.popMat();

  for (int side = -1; side <= 1; side += 2) {
    mb.pushMat();
    mb.translate(side * 0.235f, -0.46f, 0.01f);
    mb.rotateZ(-side * 0.35f);
    mb.scale(1.1f, 0.45f, 1.15f);
    mb.addEllipsoid(0.088f, 1, 1, 1, 8, 5, goldCol, PI);
    mb.popMat();

    mb.pushMat();
    mb.translate(side * 0.105f, -0.34f, 0.168f);
    mb.scale(1.15f, 1.15f, 0.42f);
    mb.addEllipsoid(0.056f, 1, 1, 1, 6, 4, goldCol, PI);
    mb.popMat();

    mb.pushMat();
    mb.translate(side * 0.285f, -0.43f, 0.01f);
    mb.rotateZ(-side * 0.24f);
    mb.rotateX(-0.14f);
    mb.addEllipsoid(0.072f, 1, 1, 1, 6, 5, skinCol, PI);

    mb.pushMat();
    mb.translate(0.0f, 0.155f, 0.0f);
    mb.addCylinder(0.064f, 0.055f, 0.31f, 8, skinCol);
    mb.popMat();

    mb.pushMat();
    mb.translate(0.0f, 0.165f, 0.0f);
    mb.addCylinder(0.067f, 0.066f, 0.052f, 10, color(28, 26, 30));
    mb.addTorusRing(0.067f, 0.007f,  0.024f, 10, 5, goldRich);
    mb.addTorusRing(0.067f, 0.007f, -0.024f, 10, 5, goldRich);
    mb.translate(side * 0.066f, 0.0f, 0.0f);
    mb.addEllipsoid(0.018f, 1, 1, 1, 6, 4, goldRich, PI);
    mb.popMat();

    mb.pushMat();
    mb.translate(0.0f, 0.31f, 0.0f);
    mb.rotateX(-0.26f);
    mb.rotateZ(side * 0.08f);

    mb.pushMat();
    mb.translate(0.0f, 0.145f, 0.0f);
    mb.addCylinder(0.053f, 0.041f, 0.29f, 8, skinCol);
    mb.popMat();

    mb.pushMat();
    mb.translate(0.0f, 0.265f, 0.0f);
    mb.addTorusRing(0.044f, 0.008f, 0.0f, 10, 5, goldRich);
    mb.popMat();

    mb.pushMat();
    mb.translate(0.0f, 0.325f, 0.012f);
    mb.scale(0.88f, 1.15f, 1.05f);
    mb.addEllipsoid(0.044f, 1, 1, 1, 6, 4, skinCol, PI);
    mb.popMat();

    mb.popMat();
    mb.popMat();
  }

  mb.pushMat();
  mb.translate(0.0f, -0.54f, 0.015f);
  mb.addCylinder(0.064f, 0.074f, 0.12f, 8, skinCol);
  mb.popMat();

  mb.pushMat();
  mb.translate(0.0f, -0.51f, 0.02f);
  mb.addTorusRing(0.082f, 0.010f, 0.0f, 12, 5, goldRich);
  mb.popMat();

  mb.popMat();
  meshManBodyStatic = mb.finish();

  PVector[] manArmorProfile = {
    new PVector(0.000f, 0.00f),
    new PVector(0.202f, 0.00f),
    new PVector(0.194f, 0.14f),
    new PVector(0.228f, 0.32f),
    new PVector(0.245f, 0.45f),
    new PVector(0.115f, 0.52f),
    new PVector(0.000f, 0.52f)
  };
  shapeManArmorTorso = buildLatheShape(manArmorProfile, 18, texManArmor, color(255, 245, 224), false);

  MeshBuilder mh = new MeshBuilder(null, color(255));
  mh.addEllipsoid(0.112f, 0.88f, 1.10f, 0.98f, 14, 10, skinCol, PI);

  mh.pushMat();
  mh.translate(0.0f, 0.006f, 0.106f);
  mh.addBox(0.020f, 0.032f, 0.026f, skinCol);
  mh.popMat();

  for (int side = -1; side <= 1; side += 2) {
    mh.pushMat();
    mh.translate(side * 0.036f, -0.016f, 0.098f);
    mh.scale(1.25f, 0.72f, 0.65f);
    mh.addEllipsoid(0.011f, 1, 1, 1, 6, 4, color(22, 16, 14), PI);
    mh.popMat();

    mh.pushMat();
    mh.translate(side * 0.037f, -0.031f, 0.101f);
    mh.rotateZ(side * 0.08f);
    mh.addBox(0.034f, 0.006f, 0.010f, hairCol);
    mh.popMat();

    mh.pushMat();
    mh.translate(side * 0.024f, 0.032f, 0.101f);
    mh.rotateZ(side * 0.24f);
    mh.scale(1.65f, 0.58f, 0.75f);
    mh.addEllipsoid(0.018f, 1, 1, 1, 6, 4, hairCol, PI);
    mh.popMat();
  }

  int ashCol = color(240, 236, 228);
  for (int line = 0; line < 3; line++) {
    float ly = -0.046f - line * 0.011f;
    mh.pushMat();
    mh.translate(0.0f, ly, 0.098f - line * 0.004f);
    mh.addBox(0.066f, 0.0042f, 0.012f, ashCol);
    mh.popMat();
  }
  mh.pushMat();
  mh.translate(0.0f, -0.057f, 0.099f);
  mh.addEllipsoid(0.0055f, 1, 1, 1, 6, 4, color(214, 34, 28), PI);
  mh.popMat();

  mh.pushMat();
  mh.translate(0.0f, 0.068f, 0.048f);
  mh.scale(0.96f, 1.15f, 0.92f);
  mh.addEllipsoid(0.104f, 0.92f, 1.05f, 0.96f, 10, 6, hairCol, PI * 0.62f);
  mh.popMat();

  appendManFlowingHair(mh, hairCol);

  mh.pushMat();
  mh.translate(0.0f, -0.135f, -0.015f);
  mh.scale(0.85f, 1.25f, 0.85f);
  mh.addEllipsoid(0.044f, 1, 1, 1, 8, 6, hairCol, PI);
  mh.popMat();

  mh.pushMat();
  mh.translate(0.0f, -0.108f, -0.015f);
  mh.addTorusRing(0.042f, 0.006f, 0.0f, 10, 5, goldRich);
  mh.popMat();

  meshManHeadFull = mh.finish();
}

void appendManFlowingHair(MeshBuilder mh, int col) {
  int segU = 10;
  int segV = 8;
  mh.ensureSpace(segU * segV * 6);
  PMatrix3D nm = mh.getNormalMatrix();

  for (int iv = 0; iv < segV; iv++) {
    float v0 = iv / (float) segV;
    float v1 = (iv + 1) / (float) segV;
    for (int iu = 0; iu < segU; iu++) {
      float u0 = iu / (float) segU;
      float u1 = (iu + 1) / (float) segU;

      float ang0 = lerp(PI * 0.42f, PI * 1.58f, u0);
      float ang1 = lerp(PI * 0.42f, PI * 1.58f, u1);
      float sA0 = sin(ang0), cA0 = cos(ang0);
      float sA1 = sin(ang1), cA1 = cos(ang1);

      float w00 = sin(v0 * 14.0f + u0 * 9.0f) * 0.012f * v0;
      float w10 = sin(v0 * 14.0f + u1 * 9.0f) * 0.012f * v0;
      float w01 = sin(v1 * 14.0f + u0 * 9.0f) * 0.012f * v1;
      float w11 = sin(v1 * 14.0f + u1 * 9.0f) * 0.012f * v1;

      float y0 = -(0.105f - v0 * 0.365f);
      float y1 = -(0.105f - v1 * 0.365f);

      float ax = sA0 * (0.112f + v0 * 0.055f + w00), az = cA0 * (0.114f + v0 * 0.032f + w00) - 0.015f;
      float bx = sA1 * (0.112f + v0 * 0.055f + w10), bz = cA1 * (0.114f + v0 * 0.032f + w10) - 0.015f;
      float cx = sA0 * (0.112f + v1 * 0.055f + w01), cz = cA0 * (0.114f + v1 * 0.032f + w01) - 0.015f;
      float dx = sA1 * (0.112f + v1 * 0.055f + w11), dz = cA1 * (0.114f + v1 * 0.032f + w11) - 0.015f;

      mh.addTriRaw(nm, ax, y0, az, bx, y0, bz, dx, y1, dz, col);
      mh.addTriRaw(nm, ax, y0, az, dx, y1, dz, cx, y1, cz, col);
    }
  }
}

// Generic Lathe Builder (SAFE: 3-arg vertex when tex == null, 5-arg when tex != null)
PShape buildLatheShape(PVector[] profile, int segs, PImage tex, int col, boolean planarUV) {
  PShape s = createShape();
  s.beginShape(TRIANGLES);
  s.noStroke();
  if (tex != null) {
    s.textureMode(NORMAL);
    s.texture(tex);
  }
  s.fill(col);
  appendLatheToShape(s, profile, segs, tex != null, planarUV, 0.0f);
  s.endShape();
  return s;
}

void appendLatheToShape(PShape s, PVector[] profile, int segs, boolean hasTex, boolean planarUV, float yOffset) {
  float minY = 99999, maxY = -99999, maxR = 0.001f;
  for (int i = 0; i < profile.length; i++) {
    minY = min(minY, profile[i].y);
    maxY = max(maxY, profile[i].y);
    maxR = max(maxR, profile[i].x);
  }
  float spanY = max(maxY - minY, 0.001f);

  float[] cosT = new float[segs + 1];
  float[] sinT = new float[segs + 1];
  float[] uT   = new float[segs + 1];
  for (int i = 0; i <= segs; i++) {
    uT[i] = i / (float) segs;
    float a = uT[i] * TWO_PI;
    cosT[i] = cos(a);
    sinT[i] = sin(a);
  }

  for (int p = 0; p < profile.length - 1; p++) {
    float r0 = profile[p].x;
    float y0 = profile[p].y;
    float r1 = profile[p + 1].x;
    float y1 = profile[p + 1].y;

    float dr = r1 - r0;
    float dy = y1 - y0;
    float nr = dy;
    float ny = -dr;
    float nLen = sqrt(nr * nr + ny * ny);
    if (nLen > 0.00001f) { nr /= nLen; ny /= nLen; }
    else { nr = 1.0f; ny = 0.0f; }

    float v0 = 1.0f - (y0 - minY) / spanY;
    float v1 = 1.0f - (y1 - minY) / spanY;
    float wy0 = -y0 + yOffset;
    float wy1 = -y1 + yOffset;

    for (int i = 0; i < segs; i++) {
      float u0 = uT[i], u1 = uT[i + 1];
      float c0 = cosT[i], s0 = sinT[i];
      float c1 = cosT[i + 1], s1 = sinT[i + 1];

      float x00 = c0 * r0, z00 = s0 * r0;
      float x10 = c1 * r0, z10 = s1 * r0;
      float x01 = c0 * r1, z01 = s0 * r1;
      float x11 = c1 * r1, z11 = s1 * r1;

      if (hasTex) {
        float tu00 = planarUV ? 0.5f + 0.5f * (x00 / maxR) : u0;
        float tv00 = planarUV ? 0.5f + 0.5f * (z00 / maxR) : v0;
        float tu10 = planarUV ? 0.5f + 0.5f * (x10 / maxR) : u1;
        float tv10 = planarUV ? 0.5f + 0.5f * (z10 / maxR) : v0;
        float tu01 = planarUV ? 0.5f + 0.5f * (x01 / maxR) : u0;
        float tv01 = planarUV ? 0.5f + 0.5f * (z01 / maxR) : v1;
        float tu11 = planarUV ? 0.5f + 0.5f * (x11 / maxR) : u1;
        float tv11 = planarUV ? 0.5f + 0.5f * (z11 / maxR) : v1;

        s.normal(c0 * nr, -ny, s0 * nr); s.vertex(x00, wy0, z00, tu00, tv00);
        s.normal(c1 * nr, -ny, s1 * nr); s.vertex(x10, wy0, z10, tu10, tv10);
        s.normal(c1 * nr, -ny, s1 * nr); s.vertex(x11, wy1, z11, tu11, tv11);

        s.normal(c0 * nr, -ny, s0 * nr); s.vertex(x00, wy0, z00, tu00, tv00);
        s.normal(c1 * nr, -ny, s1 * nr); s.vertex(x11, wy1, z11, tu11, tv11);
        s.normal(c0 * nr, -ny, s0 * nr); s.vertex(x01, wy1, z01, tu01, tv01);
      } else {
        s.normal(c0 * nr, -ny, s0 * nr); s.vertex(x00, wy0, z00);
        s.normal(c1 * nr, -ny, s1 * nr); s.vertex(x10, wy0, z10);
        s.normal(c1 * nr, -ny, s1 * nr); s.vertex(x11, wy1, z11);

        s.normal(c0 * nr, -ny, s0 * nr); s.vertex(x00, wy0, z00);
        s.normal(c1 * nr, -ny, s1 * nr); s.vertex(x11, wy1, z11);
        s.normal(c0 * nr, -ny, s0 * nr); s.vertex(x01, wy1, z01);
      }
    }
  }
}

// Appends a sculpted 3D Closed Curved Lotus Petal
void appendClosedCurvedLotusPetal(MeshBuilder mb, float height, float maxWidth, float outwardBulge,
                                  float topOffset, float tipRecurve, float cupDepth,
                                  int segU, int segV) {
  mb.ensureSpace(segU * segV * 6);
  int cols = segU + 1;
  int rows = segV + 1;
  int total = rows * cols;
  float[] px = new float[total];
  float[] py = new float[total];
  float[] pz = new float[total];
  float[] nx = new float[total];
  float[] ny = new float[total];
  float[] nz = new float[total];
  PMatrix3D nm = mb.getNormalMatrix();

  for (int iv = 0; iv <= segV; iv++) {
    float v = iv / (float) segV;
    float sinVPI = sin(v * PI);
    float widthEnv = sin(pow(max(0.0f, v), 0.72f) * PI) * (1.0f - 0.12f * v);
    float halfW = maxWidth * 0.5f * max(widthEnv, 0.002f);

    float tipT = max(0.0f, (v - 0.78f) / 0.22f);
    float radialOffset = sinVPI * outwardBulge + pow(max(0.0f, v), 1.65f) * topOffset + pow(tipT, 2.0f) * tipRecurve;
    float y = -v * height;
    int rowOff = iv * cols;

    for (int iu = 0; iu <= segU; iu++) {
      float u = iu / (float) segU;
      float s = (u - 0.5f) * 2.0f;
      float s2 = s * s;
      float concave = (1.0f - s2) * cupDepth * sinVPI;
      float midrib = exp(-s2 * 42.0f) * 0.14f * (1.0f - v * 0.5f);
      int idx = rowOff + iu;
      px[idx] = s * halfW;
      py[idx] = y;
      pz[idx] = radialOffset + concave + midrib;
    }
  }

  for (int iv = 0; iv <= segV; iv++) {
    int ivD = max(0, iv - 1) * cols;
    int ivU = min(segV, iv + 1) * cols;
    int rowOff = iv * cols;
    for (int iu = 0; iu <= segU; iu++) {
      int iuL = max(0, iu - 1);
      int iuR = min(segU, iu + 1);
      int iL = rowOff + iuL, iR = rowOff + iuR;
      int iD = ivD + iu, iU = ivU + iu;

      float dUx = px[iR] - px[iL], dUy = py[iR] - py[iL], dUz = pz[iR] - pz[iL];
      float dVx = px[iU] - px[iD], dVy = py[iU] - py[iD], dVz = pz[iU] - pz[iD];

      float cx = dVy * dUz - dVz * dUy;
      float cy = dVz * dUx - dVx * dUz;
      float cz = dVx * dUy - dVy * dUx;
      float lenSq = cx * cx + cy * cy + cz * cz;
      int idx = rowOff + iu;
      if (lenSq < 1e-8f) {
        nx[idx] = 0.0f; ny[idx] = 0.0f; nz[idx] = 1.0f;
      } else {
        float invLen = 1.0f / sqrt(lenSq);
        nx[idx] = cx * invLen; ny[idx] = cy * invLen; nz[idx] = cz * invLen;
      }
    }
  }

  for (int iv = 0; iv < segV; iv++) {
    float v0 = 1.0f - iv / (float) segV;
    float v1 = 1.0f - (iv + 1) / (float) segV;
    int r0 = iv * cols;
    int r1 = (iv + 1) * cols;
    for (int iu = 0; iu < segU; iu++) {
      float u0 = iu / (float) segU;
      float u1 = (iu + 1) / (float) segU;
      int iA = r0 + iu, iB = r0 + iu + 1;
      int iC = r1 + iu, iD = r1 + iu + 1;

      mb.emitVTex(nm, px[iA], py[iA], pz[iA], nx[iA], ny[iA], nz[iA], u0, v0);
      mb.emitVTex(nm, px[iB], py[iB], pz[iB], nx[iB], ny[iB], nz[iB], u1, v0);
      mb.emitVTex(nm, px[iD], py[iD], pz[iD], nx[iD], ny[iD], nz[iD], u1, v1);

      mb.emitVTex(nm, px[iA], py[iA], pz[iA], nx[iA], ny[iA], nz[iA], u0, v0);
      mb.emitVTex(nm, px[iD], py[iD], pz[iD], nx[iD], ny[iD], nz[iD], u1, v1);
      mb.emitVTex(nm, px[iC], py[iC], pz[iC], nx[iC], ny[iC], nz[iC], u0, v1);
    }
  }
}

// Appends a sculpted 3D Lanceolate Lower Rotor Lotus Blade
void appendLowerRotorBlade(MeshBuilder mb, float length, float maxWidth, float camber, int segU, int segV) {
  mb.ensureSpace(segU * segV * 6);
  int cols = segU + 1;
  int rows = segV + 1;
  int total = rows * cols;
  float[] px = new float[total];
  float[] py = new float[total];
  float[] pz = new float[total];
  float[] nx = new float[total];
  float[] ny = new float[total];
  float[] nz = new float[total];
  PMatrix3D nm = mb.getNormalMatrix();

  for (int iv = 0; iv <= segV; iv++) {
    float v = iv / (float) segV;
    float sinVPI = sin(v * PI);
    float widthEnv = sin(pow(max(0.0f, v), 0.68f) * PI) * (1.05f - 0.15f * v);
    float halfW = maxWidth * 0.5f * max(widthEnv, 0.003f);
    float r = 2.0f + v * length;
    float yCurve = -sin(v * PI * 0.85f) * 0.45f + pow(max(0.0f, v), 2.2f) * 0.25f;
    int rowOff = iv * cols;

    for (int iu = 0; iu <= segU; iu++) {
      float u = iu / (float) segU;
      float s = (u - 0.5f) * 2.0f;
      float s2 = s * s;
      float x = s * halfW;
      float arch = (1.0f - s2) * camber * sinVPI;
      float pitchTilt = x * 0.16f;
      float ridge = exp(-s2 * 36.0f) * 0.11f * (1.0f - v * 0.4f);
      int idx = rowOff + iu;
      px[idx] = x;
      py[idx] = -(yCurve + arch + pitchTilt + ridge);
      pz[idx] = r;
    }
  }

  for (int iv = 0; iv <= segV; iv++) {
    int ivD = max(0, iv - 1) * cols;
    int ivU = min(segV, iv + 1) * cols;
    int rowOff = iv * cols;
    for (int iu = 0; iu <= segU; iu++) {
      int iuL = max(0, iu - 1);
      int iuR = min(segU, iu + 1);
      int iL = rowOff + iuL, iR = rowOff + iuR;
      int iD = ivD + iu, iU = ivU + iu;

      float dUx = px[iR] - px[iL], dUy = py[iR] - py[iL], dUz = pz[iR] - pz[iL];
      float dVx = px[iU] - px[iD], dVy = py[iU] - py[iD], dVz = pz[iU] - pz[iD];

      float cx = dUy * dVz - dUz * dVy;
      float cy = dUz * dVx - dUx * dVz;
      float cz = dUx * dVy - dUy * dVx;
      float lenSq = cx * cx + cy * cy + cz * cz;
      int idx = rowOff + iu;
      if (lenSq < 1e-8f) {
        nx[idx] = 0.0f; ny[idx] = -1.0f; nz[idx] = 0.0f;
      } else {
        float invLen = 1.0f / sqrt(lenSq);
        nx[idx] = cx * invLen; ny[idx] = cy * invLen; nz[idx] = cz * invLen;
      }
    }
  }

  for (int iv = 0; iv < segV; iv++) {
    float v0 = 1.0f - iv / (float) segV;
    float v1 = 1.0f - (iv + 1) / (float) segV;
    int r0 = iv * cols;
    int r1 = (iv + 1) * cols;
    for (int iu = 0; iu < segU; iu++) {
      float u0 = iu / (float) segU;
      float u1 = (iu + 1) / (float) segU;
      int iA = r0 + iu, iB = r0 + iu + 1;
      int iC = r1 + iu, iD = r1 + iu + 1;

      mb.emitVTex(nm, px[iA], py[iA], pz[iA], nx[iA], ny[iA], nz[iA], u0, v0);
      mb.emitVTex(nm, px[iD], py[iD], pz[iD], nx[iD], ny[iD], nz[iD], u1, v1);
      mb.emitVTex(nm, px[iB], py[iB], pz[iB], nx[iB], ny[iB], nz[iB], u1, v0);

      mb.emitVTex(nm, px[iA], py[iA], pz[iA], nx[iA], ny[iA], nz[iA], u0, v0);
      mb.emitVTex(nm, px[iC], py[iC], pz[iC], nx[iC], ny[iC], nz[iC], u0, v1);
      mb.emitVTex(nm, px[iD], py[iD], pz[iD], nx[iD], ny[iD], nz[iD], u1, v1);
    }
  }
}

// Golden-Hour Sky Dome Sphere
PShape buildSkyDome(float radius, int segU, int segV) {
  PVector sunDir = new PVector(0.68f, 0.34f, -0.65f).normalize();
  int topCol = color(40, 75, 122);
  int midCol = color(200, 139, 84);
  int horizonCol = color(255, 224, 163);
  int bottomCol = color(158, 116, 72);

  float[] cosP = new float[segU + 1];
  float[] sinP = new float[segU + 1];
  for (int iu = 0; iu <= segU; iu++) {
    float p = (iu / (float) segU) * TWO_PI;
    cosP[iu] = cos(p);
    sinP[iu] = sin(p);
  }

  PShape s = createShape();
  s.beginShape(TRIANGLES);
  s.noStroke();

  for (int iv = 0; iv < segV; iv++) {
    float t0 = (iv / (float) segV) * PI;
    float t1 = ((iv + 1) / (float) segV) * PI;
    float st0 = sin(t0), ct0 = cos(t0);
    float st1 = sin(t1), ct1 = cos(t1);
    for (int iu = 0; iu < segU; iu++) {
      float cp0 = cosP[iu], sp0 = sinP[iu];
      float cp1 = cosP[iu + 1], sp1 = sinP[iu + 1];

      float d00x = st0 * cp0, d00y = ct0, d00z = st0 * sp0;
      float d10x = st0 * cp1, d10y = ct0, d10z = st0 * sp1;
      float d01x = st1 * cp0, d01y = ct1, d01z = st1 * sp0;
      float d11x = st1 * cp1, d11y = ct1, d11z = st1 * sp1;

      int c00 = evalSkyColor(d00x, d00y, d00z, sunDir, topCol, midCol, horizonCol, bottomCol);
      int c10 = evalSkyColor(d10x, d10y, d10z, sunDir, topCol, midCol, horizonCol, bottomCol);
      int c01 = evalSkyColor(d01x, d01y, d01z, sunDir, topCol, midCol, horizonCol, bottomCol);
      int c11 = evalSkyColor(d11x, d11y, d11z, sunDir, topCol, midCol, horizonCol, bottomCol);

      s.fill(c00); s.vertex(d00x * radius, -d00y * radius, d00z * radius);
      s.fill(c11); s.vertex(d11x * radius, -d11y * radius, d11z * radius);
      s.fill(c10); s.vertex(d10x * radius, -d10y * radius, d10z * radius);

      s.fill(c00); s.vertex(d00x * radius, -d00y * radius, d00z * radius);
      s.fill(c01); s.vertex(d01x * radius, -d01y * radius, d01z * radius);
      s.fill(c11); s.vertex(d11x * radius, -d11y * radius, d11z * radius);
    }
  }
  s.endShape();
  return s;
}

int evalSkyColor(float dx, float dy, float dz, PVector sunDir, int topCol, int midCol, int horizonCol, int bottomCol) {
  int col;
  if (dy >= 0.0f) {
    float t1 = smoothstep(0.0f, 0.24f, dy);
    float t2 = smoothstep(0.18f, 0.85f, dy);
    col = lerpColor(horizonCol, midCol, t1);
    col = lerpColor(col, topCol, t2);
  } else {
    float tb = smoothstep(0.0f, 0.45f, -dy);
    col = lerpColor(horizonCol, bottomCol, tb);
  }
  float sunAmt = max(0.0f, dx * sunDir.x + dy * sunDir.y + dz * sunDir.z);
  float glow = pow(sunAmt, 14.0f) * 0.45f + pow(sunAmt, 120.0f) * 1.1f;
  float r = min(255, red(col) + 255 * glow);
  float g = min(255, green(col) + 220 * glow);
  float b = min(255, blue(col) + 145 * glow);
  return color(r, g, b);
}

// Multi-Octave Lush Green Mountain Valley Terrain (48x48x6 = 13,824 vertices)
PShape buildTerrainMesh(float size, int segs) {
  int cols = segs + 1;
  int total = cols * cols;
  float[] px = new float[total];
  float[] py = new float[total];
  float[] pz = new float[total];
  int[] colsGrid = new int[total];

  int valleyCol = color(31, 61, 36);
  int forestCol = color(59, 99, 50);
  int ridgeCol = color(124, 128, 82);
  float invSegs = 1.0f / segs;

  for (int iz = 0; iz <= segs; iz++) {
    float vz = (iz * invSegs - 0.5f) * size;
    float cosZ8 = cos(vz * 0.008f);
    float sinZ19 = sin(vz * 0.019f);
    int rowOff = iz * cols;
    for (int ix = 0; ix <= segs; ix++) {
      float vx = (ix * invSegs - 0.5f) * size;
      float h1 = sin(vx * 0.008f) * cosZ8 * 34.0f;
      float h2 = sin(vx * 0.021f + 1.3f) * sinZ19 * 16.0f;
      float h3 = cos(vx * 0.048f - vz * 0.042f) * 6.5f;
      float ridge = abs(sin(vx * 0.006f + vz * 0.005f)) * 28.0f;
      float vy = h1 + h2 + h3 + ridge;
      int idx = rowOff + ix;
      px[idx] = vx;
      py[idx] = -vy;
      pz[idx] = vz;

      float normH = constrain((vy + 35.0f) / 95.0f, 0.0f, 1.0f);
      if (normH < 0.55f) {
        colsGrid[idx] = lerpColor(valleyCol, forestCol, normH * 1.8181819f);
      } else {
        colsGrid[idx] = lerpColor(forestCol, ridgeCol, (normH - 0.55f) * 2.2222223f);
      }
    }
  }

  PShape s = createShape();
  s.beginShape(TRIANGLES);
  s.noStroke();
  for (int iz = 0; iz < segs; iz++) {
    int r0 = iz * cols;
    int r1 = (iz + 1) * cols;
    for (int ix = 0; ix < segs; ix++) {
      int iA = r0 + ix, iB = r0 + ix + 1;
      int iC = r1 + ix, iD = r1 + ix + 1;

      float acx = px[iC] - px[iA], acy = py[iC] - py[iA], acz = pz[iC] - pz[iA];
      float abx = px[iB] - px[iA], aby = py[iB] - py[iA], abz = pz[iB] - pz[iA];
      float n1x = acy * abz - acz * aby;
      float n1y = acz * abx - acx * abz;
      float n1z = acx * aby - acy * abx;
      float len1 = sqrt(n1x * n1x + n1y * n1y + n1z * n1z);
      if (len1 > 1e-8f) { float inv1 = 1.0f / len1; n1x *= inv1; n1y *= inv1; n1z *= inv1; }
      else { n1x = 0.0f; n1y = -1.0f; n1z = 0.0f; }

      s.normal(n1x, n1y, n1z);
      s.fill(colsGrid[iA]); s.vertex(px[iA], py[iA], pz[iA]);
      s.fill(colsGrid[iC]); s.vertex(px[iC], py[iC], pz[iC]);
      s.fill(colsGrid[iB]); s.vertex(px[iB], py[iB], pz[iB]);

      float dbx = px[iD] - px[iB], dby = py[iD] - py[iB], dbz = pz[iD] - pz[iB];
      float cbx = px[iC] - px[iB], cby = py[iC] - py[iB], cbz = pz[iC] - pz[iB];
      float n2x = dby * cbz - dbz * cby;
      float n2y = dbz * cbx - dbx * cbz;
      float n2z = dbx * cby - dby * cbx;
      float len2 = sqrt(n2x * n2x + n2y * n2y + n2z * n2z);
      if (len2 > 1e-8f) { float inv2 = 1.0f / len2; n2x *= inv2; n2y *= inv2; n2z *= inv2; }
      else { n2x = 0.0f; n2y = -1.0f; n2z = 0.0f; }

      s.normal(n2x, n2y, n2z);
      s.fill(colsGrid[iB]); s.vertex(px[iB], py[iB], pz[iB]);
      s.fill(colsGrid[iC]); s.vertex(px[iC], py[iC], pz[iC]);
      s.fill(colsGrid[iD]); s.vertex(px[iD], py[iD], pz[iD]);
    }
  }
  s.endShape();
  return s;
}

// ============================================================================
// 8. DIRECT ARGB PROCEDURAL TEXTURE GENERATORS (ZERO PGRAPHICS / ZERO OOM)
// ============================================================================
PImage createUpperLotusPetalTexture() {
  int w = 256, h = 256;
  PImage img = createImage(w, h, ARGB);
  img.loadPixels();

  int c0 = color(198, 147, 61);
  int c1 = color(237, 217, 180);
  int c2 = color(251, 246, 236);
  int c3 = color(245, 235, 216);
  int c4 = color(214, 164, 73);
  int veinCol = color(175, 130, 56);
  int borderOuter = color(200, 140, 40);
  int borderInner = color(255, 225, 140);
  int midribCol = color(245, 212, 118);

  for (int y = 0; y < h; y++) {
    float t = 1.0f - y / (float) h;
    int baseCol;
    if (t < 0.16f) baseCol = lerpColor(c0, c1, t / 0.16f);
    else if (t < 0.48f) baseCol = lerpColor(c1, c2, (t - 0.16f) / 0.32f);
    else if (t < 0.82f) baseCol = lerpColor(c2, c3, (t - 0.48f) / 0.34f);
    else baseCol = lerpColor(c3, c4, (t - 0.82f) / 0.18f);

    float spread = max(0.05f, sin(pow(max(0.01f, t), 0.72f) * PI));
    float invSpread26 = 26.0f / spread;
    float tShift = t * 6.0f;
    float midribHalfW = 0.016f * (1.0f - t * 0.6f);
    boolean yOuter = (y < 7 || y >= h - 7);
    boolean yInner = (y < 10 || y >= h - 10);
    int rowOff = y * w;

    for (int x = 0; x < w; x++) {
      if (yOuter || x < 7 || x >= w - 7) {
        img.pixels[rowOff + x] = borderOuter;
        continue;
      }
      if (yInner || x < 10 || x >= w - 10) {
        img.pixels[rowOff + x] = borderInner;
        continue;
      }

      float ax = abs((x - w * 0.5f) / (w * 0.5f));
      if (ax < midribHalfW) {
        img.pixels[rowOff + x] = midribCol;
        continue;
      }

      int col = baseCol;
      float vein = abs(sin((ax * invSpread26 - tShift) * PI));
      if (vein > 0.94f && ax < spread) {
        col = lerpColor(col, veinCol, 0.38f);
      }
      img.pixels[rowOff + x] = col;
    }
  }
  img.updatePixels();
  return img;
}

PImage createDomeFiligreeTexture() {
  int w = 256, h = 256;
  PImage img = createImage(w, h, ARGB);
  img.loadPixels();

  int g0 = color(240, 190, 89);
  int g1 = color(217, 155, 46);
  int g2 = color(199, 135, 34);
  int g3 = color(156, 99, 20);
  int darkLine = color(107, 66, 8);
  int brightLine = color(255, 229, 156);
  float cellW = w / 12.0f;
  float rOuter = cellW * 0.41f;
  float rInnerSq = (cellW * 0.12f) * (cellW * 0.12f);
  float rMedMaxSq = (cellW * 0.36f) * (cellW * 0.36f);

  for (int y = 0; y < h; y++) {
    float t = y / (float) h;
    int baseCol;
    if (t < 0.3f) baseCol = lerpColor(g0, g1, t / 0.3f);
    else if (t < 0.65f) baseCol = lerpColor(g1, g2, (t - 0.3f) / 0.35f);
    else baseCol = lerpColor(g2, g3, (t - 0.65f) / 0.35f);

    boolean isRingDark = (abs(y - 22) < 3 || abs(y - 58) < 3 || abs(y - 98) < 3 ||
                          abs(y - 140) < 3 || abs(y - 185) < 3 || abs(y - 222) < 3 || abs(y - 244) < 3);
    boolean isRingBright = (abs(y - 22) < 1 || abs(y - 58) < 1 || abs(y - 98) < 1 ||
                            abs(y - 140) < 1 || abs(y - 185) < 1 || abs(y - 222) < 1 || abs(y - 244) < 1);
    boolean isChevronBand = (y > 26 && y < 94);
    boolean isMedallionBand = ((y > 103 && y < 135) || (y > 146 && y < 178));
    float medCy = (y < 140) ? 119.0f : 162.0f;

    int rowOff = y * w;
    for (int x = 0; x < w; x++) {
      if (isRingBright) { img.pixels[rowOff + x] = brightLine; continue; }
      if (isRingDark)   { img.pixels[rowOff + x] = darkLine;   continue; }

      int col = baseCol;
      if (isChevronBand) {
        float wave = abs(((x % 22) / 11.0f) - 1.0f);
        float targetY = (y < 58) ? lerp(28, 53, wave) : lerp(62, 92, wave);
        if (abs(y - targetY) < 2.2f) col = (y < 58) ? darkLine : brightLine;
      } else if (isMedallionBand) {
        float cx = (floor(x / cellW) + 0.5f) * cellW;
        float dx = x - cx;
        float dy = y - medCy;
        float dSq = dx * dx + dy * dy;
        if (dSq < rMedMaxSq) {
          if (dSq > rInnerSq && abs(sin(atan2(dy, dx) * 4.0f)) > 0.72f) {
            col = lerpColor(col, brightLine, 0.45f);
          }
        } else if (abs(sqrt(dSq) - rOuter) < 1.3f) {
          col = darkLine;
        }
      }

      img.pixels[rowOff + x] = col;
    }
  }
  img.updatePixels();
  return img;
}

PImage createLowerRotorBladeTexture() {
  int w = 128, h = 256;
  PImage img = createImage(w, h, ARGB);
  img.loadPixels();

  int b0 = color(120, 76, 20);
  int b1 = color(184, 125, 40);
  int b2 = color(219, 160, 59);
  int b3 = color(245, 199, 103);
  int darkFeather = color(65, 38, 8);
  int ridgeCol = color(247, 207, 116);
  int edgeCol = color(255, 223, 142);

  for (int y = 0; y < h; y++) {
    float t = 1.0f - y / (float) h;
    int baseCol;
    if (t < 0.35f) baseCol = lerpColor(b0, b1, t / 0.35f);
    else if (t < 0.75f) baseCol = lerpColor(b1, b2, (t - 0.35f) / 0.40f);
    else baseCol = lerpColor(b2, b3, (t - 0.75f) / 0.25f);

    boolean yEdge = (y < 3 || y >= h - 3);
    int rowOff = y * w;
    for (int x = 0; x < w; x++) {
      if (yEdge || x < 3 || x >= w - 3) {
        img.pixels[rowOff + x] = edgeCol;
        continue;
      }
      float nx = abs((x - w * 0.5f) / (w * 0.5f));
      if (nx < 0.04f) {
        img.pixels[rowOff + x] = ridgeCol;
        continue;
      }

      int col = baseCol;
      float feather = sin((t * 32.0f - nx * 4.5f) * PI);
      if (feather > 0.75f) col = lerpColor(col, darkFeather, 0.35f);
      else if (feather < -0.75f) col = lerpColor(col, edgeCol, 0.32f);

      img.pixels[rowOff + x] = col;
    }
  }
  img.updatePixels();
  return img;
}

PImage createDeckFloorTexture() {
  int size = 256;
  PImage img = createImage(size, size, ARGB);
  img.loadPixels();

  float half = size * 0.5f;
  int bgCol = color(143, 93, 26);
  int goldLight = color(242, 195, 99);
  int goldMid = color(212, 152, 49);
  int crimson = color(114, 38, 25);

  for (int y = 0; y < size; y++) {
    float dy = (y - half) / half;
    float dy2 = dy * dy;
    int rowOff = y * size;
    for (int x = 0; x < size; x++) {
      float dx = (x - half) / half;
      float r2 = dx * dx + dy2;
      int col = bgCol;

      if (r2 > 0.8464f) col = goldMid;
      else if (r2 > 0.7056f) col = crimson;
      else if (r2 > 0.6400f) col = goldLight;
      else if (r2 > 0.3844f) {
        col = goldMid;
        float spoke = abs(sin(atan2(dy, dx) * 12.0f));
        if (spoke > 0.92f) col = goldLight;
        else if (spoke < 0.45f && abs(sqrt(r2) - 0.71f) < 0.04f) {
          col = lerpColor(goldMid, goldLight, 0.45f);
        }
      } else if (r2 > 0.2809f) col = crimson;
      else if (r2 > 0.2116f) col = goldLight;

      img.pixels[rowOff + x] = col;
    }
  }
  img.updatePixels();
  return img;
}

PImage createCloudTexture() {
  int size = 128;
  PImage img = createImage(size, size, ARGB);
  img.loadPixels();

  float cx = size * 0.5f, cy = size * 0.48f;
  float invHalf = 1.0f / (size * 0.5f);
  int clearCloud = color(255, 232, 198, 0);

  for (int y = 0; y < size; y++) {
    float dy = (y - cy) * invHalf;
    float dy2 = dy * dy;
    int rowOff = y * size;
    for (int x = 0; x < size; x++) {
      float dx = (x - cx) * invHalf;
      float d2 = dx * dx + dy2;
      if (d2 < 1.0f) {
        float t = 1.0f - sqrt(d2);
        float alpha = pow(t, 1.8f) * 235.0f;
        img.pixels[rowOff + x] = color(255, lerp(215, 248, t), lerp(165, 232, t), alpha);
      } else {
        img.pixels[rowOff + x] = clearCloud;
      }
    }
  }
  img.updatePixels();
  return img;
}

PImage createManArmorTexture() {
  int w = 256, h = 256;
  PImage img = createImage(w, h, ARGB);
  img.loadPixels();

  int darkTop = color(34, 31, 36);
  int darkBot = color(20, 18, 22);
  int gold = color(236, 182, 72);
  int goldHi = color(255, 228, 148);
  float medPy = h * 0.36f;

  for (int y = 0; y < h; y++) {
    float t = y / (float) h;
    int baseCol = lerpColor(darkTop, darkBot, t);
    boolean isBandGold = (y < 17 || y >= h - 19);
    boolean isBandHi   = (!isBandGold && (y < 20 || y >= h - 22));
    boolean isAbBand   = (y > h * 0.50f && y < h * 0.86f);
    int rowOff = y * w;

    for (int x = 0; x < w; x++) {
      if (isBandGold) { img.pixels[rowOff + x] = gold;   continue; }
      if (isBandHi)   { img.pixels[rowOff + x] = goldHi; continue; }

      int col = baseCol;
      float cx = (x < w * 0.5f) ? w * 0.25f : w * 0.75f;
      float dx = x - cx;
      if (abs(dx) < 2) col = gold;

      float dyMed = y - medPy;
      float dyMed2 = dyMed * dyMed;
      if (dyMed2 < 920.0f) {
        for (int side = -1; side <= 1; side += 2) {
          float dxMed = x - (cx + side * 28.0f);
          float d2 = dxMed * dxMed + dyMed2;
          if (d2 < 36.0f) col = goldHi;
          else if (d2 < 324.0f && abs(sin(atan2(dyMed, dxMed) * 4.0f)) > 0.65f) col = gold;
          else if (d2 > 665.0f && d2 < 912.0f && y > medPy - 14) col = gold;
        }
      }

      if (isAbBand && abs(dx) < 48) {
        float ribY = ((y - h * 0.50f) - abs(dx) * 0.32f) % 21.0f;
        if (ribY >= 0 && ribY < 3.0f) col = gold;
      }

      img.pixels[rowOff + x] = col;
    }
  }
  img.updatePixels();
  return img;
}

PImage createBlurHaloTexture() {
  int size = 128;
  PImage img = createImage(size, size, ARGB);
  img.loadPixels();

  float half = size * 0.5f;
  int clearHalo = color(255, 205, 105, 0);

  for (int y = 0; y < size; y++) {
    float dy = (y - half) / half;
    float dy2 = dy * dy;
    int rowOff = y * size;
    for (int x = 0; x < size; x++) {
      float dx = (x - half) / half;
      float d2 = dx * dx + dy2;
      if (d2 >= 0.0324f && d2 <= 0.9801f) {
        float d = sqrt(d2);
        float t = (d - 0.18f) / 0.81f;
        float mask = smoothstep(0.0f, 0.22f, t) * (1.0f - smoothstep(0.65f, 1.0f, t));
        float streak = 0.65f + 0.35f * sin(atan2(dy, dx) * 16.0f + t * 3.5f);
        float a = constrain(mask * streak * 255.0f, 0, 255);
        img.pixels[rowOff + x] = color(255, 205, 105, a);
      } else {
        img.pixels[rowOff + x] = clearHalo;
      }
    }
  }
  img.updatePixels();
  return img;
}

// ============================================================================
// 9. 2D GLASSMORPHIC / GOLDEN HUD OVERLAY & TOUCH CONTROLS
// ============================================================================
void drawLoadingScreen() {
  background(18, 12, 7);
  noLights();
  camera();
  ortho();
  hint(DISABLE_DEPTH_TEST);

  float cx = width * 0.5f;
  float cy = height * 0.5f;
  float sc = min(width, height) / 420.0f;

  pushMatrix();
  translate(cx, cy - 36 * sc);
  rotate(millis() * 0.004f);
  noFill();
  stroke(251, 191, 36, 55);
  strokeWeight(3 * sc);
  ellipse(0, 0, 68 * sc, 68 * sc);
  stroke(253, 224, 130);
  arc(0, 0, 68 * sc, 68 * sc, 0, HALF_PI * 1.4f);
  popMatrix();

  noStroke();
  fill(245, 178, 52, 215);
  ellipse(cx, cy - 36 * sc, 34 * sc, 34 * sc);

  textAlign(CENTER, CENTER);
  fill(colHudGoldText);
  textSize(20 * sc);
  text("PUSHPAKA VIMANA", cx, cy + 20 * sc);

  fill(254, 243, 199, 175);
  textSize(12 * sc);
  text(loadingStatus, cx, cy + 46 * sc);

  hint(ENABLE_DEPTH_TEST);
}

void updateHudLayoutIfNeeded() {
  if (width == hudLayoutW && height == hudLayoutH) return;
  hudLayoutW = width;
  hudLayoutH = height;

  hudSc = constrain(min(width, height) / 440.0f, 0.78f, 1.45f);
  hudPad = 14.0f * hudSc;

  float hideW = 104 * hudSc;
  float hideH = 36 * hudSc;
  rectHideBtn[0] = width - hudPad - hideW;
  rectHideBtn[1] = hudPad;
  rectHideBtn[2] = hideW;
  rectHideBtn[3] = hideH;

  hudTitleW = min(width - hideW - hudPad * 3.0f, 455 * hudSc);
  hudTitleH = 54 * hudSc;

  String subTxt = "Dual Hollow Decks (Man Above • Woman Below) • 24 Closed Petals • 32 Rotor Blades";
  hudSubTextSize = 9.5f * hudSc;
  textSize(hudSubTextSize);
  float subW = textWidth(subTxt);
  float maxSubW = max(hudTitleW - 28.0f * hudSc, 60.0f);
  if (subW > maxSubW) hudSubTextSize *= (maxSubW / subW);

  boolean isLandscape = width >= height;
  hudRotorPanelW = isLandscape ? 295 * hudSc : (width - hudPad * 2.0f);
  hudRotorPanelH = 68 * hudSc;
  hudRotorX = width - hudPad - hudRotorPanelW;
  hudRotorY = height - hudPad - hudRotorPanelH;

  float smW = 78 * hudSc;
  float smH = 26 * hudSc;
  rectSlowMoBtn[0] = hudRotorX + hudRotorPanelW - 12 * hudSc - smW;
  rectSlowMoBtn[1] = hudRotorY + 32 * hudSc;
  rectSlowMoBtn[2] = smW;
  rectSlowMoBtn[3] = smH;

  hudSliderX = hudRotorX + 14 * hudSc;
  hudSliderW = rectSlowMoBtn[0] - hudSliderX - 14 * hudSc;
  hudSliderY = hudRotorY + 45 * hudSc;
  rectSliderTrack[0] = hudSliderX;
  rectSliderTrack[1] = hudSliderY - 14 * hudSc;
  rectSliderTrack[2] = hudSliderW;
  rectSliderTrack[3] = 28 * hudSc;

  hudCamPanelW = isLandscape ? (hudRotorX - hudPad * 2.0f) : (width - hudPad * 2.0f);
  hudCamPanelH = isLandscape ? 68 * hudSc : 86 * hudSc;
  hudCamPanelX = hudPad;
  hudCamPanelY = isLandscape ? (height - hudPad - hudCamPanelH) : (hudRotorY - hudPad * 0.7f - hudCamPanelH);

  int cols = isLandscape ? 4 : 2;
  int rows = isLandscape ? 1 : 2;
  float gap = 6 * hudSc;
  float btnAreaW = hudCamPanelW - 20 * hudSc;
  float btnAreaH = hudCamPanelH - 28 * hudSc;
  float btnW = (btnAreaW - gap * (cols - 1)) / cols;
  float btnH = (btnAreaH - gap * (rows - 1)) / rows;

  hudPresetTextSize = 9.5f * hudSc;
  textSize(hudPresetTextSize);
  float widestBtnTxt = 1.0f;
  for (int i = 0; i < 4; i++) {
    int c = i % cols;
    int r = i / cols;
    rectPresetBtns[i][0] = hudCamPanelX + 10 * hudSc + c * (btnW + gap);
    rectPresetBtns[i][1] = hudCamPanelY + 24 * hudSc + r * (btnH + gap);
    rectPresetBtns[i][2] = btnW;
    rectPresetBtns[i][3] = btnH;
    widestBtnTxt = max(widestBtnTxt, textWidth(presetLabels[i]));
  }
  float maxBtnTxtW = max(btnW - 10.0f * hudSc, 40.0f);
  if (widestBtnTxt > maxBtnTxtW) hudPresetTextSize *= (maxBtnTxtW / widestBtnTxt);
}

void drawHUD() {
  noLights();
  camera();
  ortho();
  hint(DISABLE_DEPTH_TEST);

  updateHudLayoutIfNeeded();
  float sc = hudSc;
  float pad = hudPad;

  // 1. Top-Right Toggle UI Button
  drawGlassPanel(rectHideBtn[0], rectHideBtn[1], rectHideBtn[2], rectHideBtn[3], 11 * sc, 175);
  fill(colHudCreamText);
  textAlign(CENTER, CENTER);
  textSize(12 * sc);
  text(uiVisible ? "Hide UI" : "Show UI", rectHideBtn[0] + rectHideBtn[2] * 0.5f, rectHideBtn[1] + rectHideBtn[3] * 0.5f);

  // 2. Top-Left Title Header Card
  drawGlassPanel(pad, pad, hudTitleW, hudTitleH, 14 * sc, uiVisible ? 175 : 70);

  fill(251, 191, 36, uiVisible ? 255 : 95);
  noStroke();
  ellipse(pad + 15 * sc, pad + 18 * sc, 7 * sc, 7 * sc);

  textAlign(LEFT, CENTER);
  fill(253, 230, 168, uiVisible ? 255 : 95);
  textSize(14 * sc);
  text("PUSHPAKA VIMANA", pad + 26 * sc, pad + 17 * sc);

  fill(254, 243, 199, uiVisible ? 185 : 70);
  textSize(hudSubTextSize);
  text("Dual Hollow Decks (Man Above • Woman Below) • 24 Closed Petals • 32 Rotor Blades", pad + 14 * sc, pad + 37 * sc);

  if (!uiVisible) {
    hint(ENABLE_DEPTH_TEST);
    return;
  }

  // 3. Bottom Control Dock (Rotor Panel)
  drawGlassPanel(hudRotorX, hudRotorY, hudRotorPanelW, hudRotorPanelH, 14 * sc, 175);

  textAlign(LEFT, CENTER);
  fill(252, 211, 77, 220);
  textSize(10 * sc);
  text("LOWER LOTUS ROTOR", hudRotorX + 12 * sc, hudRotorY + 17 * sc);

  if (abs(rotorSpeed - lastFormattedRotorSpeed) > 0.01f) {
    lastFormattedRotorSpeed = rotorSpeed;
    int rpm = round((rotorSpeed * 60.0f) / TWO_PI);
    cachedRpmLabel = nf(rotorSpeed, 1, 1) + " rad/s (" + rpm + " RPM)";
  }
  textAlign(RIGHT, CENTER);
  fill(colHudGoldText);
  textSize(11 * sc);
  text(cachedRpmLabel, hudRotorX + hudRotorPanelW - 12 * sc, hudRotorY + 17 * sc);

  // Slow-Mo Button inside Rotor Panel
  fill(isSlowMo ? colSlowMoOnFill : colSlowMoOffFill);
  stroke(253, 224, 130, isSlowMo ? 230 : 110);
  strokeWeight(1.2f * sc);
  rect(rectSlowMoBtn[0], rectSlowMoBtn[1], rectSlowMoBtn[2], rectSlowMoBtn[3], 8 * sc);

  textAlign(CENTER, CENTER);
  fill(colHudCreamText);
  textSize(10.5f * sc);
  text(isSlowMo ? "Full Speed" : "Slow-Mo",
       rectSlowMoBtn[0] + rectSlowMoBtn[2] * 0.5f,
       rectSlowMoBtn[1] + rectSlowMoBtn[3] * 0.5f);

  // Gold Range Slider Track & Thumb
  noStroke();
  fill(colSliderTrack);
  rect(hudSliderX, hudSliderY - 3 * sc, hudSliderW, 6 * sc, 3 * sc);

  float normSpeed = constrain(rotorSpeed / 26.0f, 0.0f, 1.0f);
  fill(colSliderFill);
  rect(hudSliderX, hudSliderY - 3 * sc, hudSliderW * normSpeed, 6 * sc, 3 * sc);

  float thumbX = hudSliderX + hudSliderW * normSpeed;
  fill(245, 185, 66, 95);
  ellipse(thumbX, hudSliderY, 24 * sc, 24 * sc);
  fill(255, 243, 196);
  stroke(217, 155, 38);
  strokeWeight(2 * sc);
  ellipse(thumbX, hudSliderY, 15 * sc, 15 * sc);

  // 4. Camera Presets Panel
  drawGlassPanel(hudCamPanelX, hudCamPanelY, hudCamPanelW, hudCamPanelH, 14 * sc, 175);

  textAlign(LEFT, CENTER);
  fill(252, 211, 77, 220);
  textSize(10 * sc);
  text("CAMERA VIEWS", hudCamPanelX + 12 * sc, hudCamPanelY + 15 * sc);

  textAlign(RIGHT, CENTER);
  fill(254, 243, 199, 130);
  textSize(9.5f * sc);
  text("Drag: Orbit • Pinch: Zoom", hudCamPanelX + hudCamPanelW - 12 * sc, hudCamPanelY + 15 * sc);

  textAlign(CENTER, CENTER);
  textSize(hudPresetTextSize);
  strokeWeight(1.2f * sc);
  for (int i = 0; i < 4; i++) {
    float bx = rectPresetBtns[i][0];
    float by = rectPresetBtns[i][1];
    float bw = rectPresetBtns[i][2];
    float bh = rectPresetBtns[i][3];
    boolean isAct = (i == activePreset);

    fill(isAct ? colBtnActiveFill : colBtnInactiveFill);
    stroke(isAct ? colBtnActiveStroke : colBtnInactiveStroke);
    rect(bx, by, bw, bh, 9 * sc);

    fill(isAct ? color(255, 246, 219) : color(254, 243, 199, 220));
    text(presetLabels[i], bx + bw * 0.5f, by + bh * 0.5f);
  }

  hint(ENABLE_DEPTH_TEST);
}

void drawGlassPanel(float x, float y, float w, float h, float r, float alpha) {
  fill(18, 13, 9, alpha);
  stroke(242, 193, 92, alpha * 0.45f);
  strokeWeight(1.2f);
  rect(x, y, w, h, r);
}

// ============================================================================
// 10. ANDROID MULTI-TOUCH, ORBIT CAMERA & HUD INPUT HANDLERS
// ============================================================================
void applyCameraPreset(int idx) {
  activePreset = idx;
  goalCamPos.set(presetPositions[idx]);
  goalCamTarget.set(presetTargets[idx]);
  isTransitioningCamera = true;
}

void syncSphericalFromCartesian() {
  float dx = camPos.x - camTarget.x;
  float dy = camPos.y - camTarget.y;
  float dz = camPos.z - camTarget.z;
  camDistance = constrain(sqrt(dx * dx + dy * dy + dz * dz), 1.1f, 140.0f);
  camPitch = asin(constrain(dy / camDistance, -0.96f, 0.96f));
  camYaw = atan2(dx, dz);
}

void updateCartesianFromSpherical() {
  camDistance = constrain(camDistance, 1.1f, 140.0f);
  camPitch = constrain(camPitch, -1.45f, 1.45f);
  float cosP = cos(camPitch);
  camPos.x = camTarget.x + camDistance * cosP * sin(camYaw);
  camPos.y = camTarget.y + camDistance * sin(camPitch);
  camPos.z = camTarget.z + camDistance * cosP * cos(camYaw);
}

boolean hitRect(float px, float py, float[] r) {
  return px >= r[0] && px <= r[0] + r[2] && py >= r[1] && py <= r[1] + r[3];
}

void updateSliderFromX(float px) {
  float t = constrain((px - rectSliderTrack[0]) / max(rectSliderTrack[2], 1.0f), 0.0f, 1.0f);
  rotorSpeed = t * 26.0f;
  isSlowMo = false;
}

void mousePressed() {
  if (!isInitialized) return;
  updateHudLayoutIfNeeded();
  prevTouchX = mouseX;
  prevTouchY = mouseY;
  prevPinchDist = -1;
  wasPinching = false;

  if (hitRect(mouseX, mouseY, rectHideBtn)) {
    uiVisible = !uiVisible;
    return;
  }

  if (uiVisible) {
    for (int i = 0; i < 4; i++) {
      if (hitRect(mouseX, mouseY, rectPresetBtns[i])) {
        applyCameraPreset(i);
        return;
      }
    }

    if (hitRect(mouseX, mouseY, rectSlowMoBtn)) {
      isSlowMo = !isSlowMo;
      rotorSpeed = isSlowMo ? 1.4f : 15.0f;
      return;
    }

    if (hitRect(mouseX, mouseY, rectSliderTrack)) {
      draggingSlider = true;
      updateSliderFromX(mouseX);
      return;
    }
  }

  draggingOrbit = true;
  isTransitioningCamera = false;
}

void mouseDragged() {
  if (!isInitialized) return;

  if (touches != null && touches.length >= 2) {
    draggingSlider = false;
    draggingOrbit = false;
    isTransitioningCamera = false;
    wasPinching = true;

    float x0 = touches[0].x, y0 = touches[0].y;
    float x1 = touches[1].x, y1 = touches[1].y;
    float currDist = dist(x0, y0, x1, y1);
    float midX = (x0 + x1) * 0.5f;
    float midY = (y0 + y1) * 0.5f;

    if (prevPinchDist > 0.0f && currDist > 5.0f) {
      float zoomRatio = prevPinchDist / currDist;
      camDistance = constrain(camDistance * zoomRatio, 1.1f, 140.0f);

      float scaleFactor = 0.015f * (camDistance / 12.0f);
      float dx = (midX - prevPinchMid.x) * scaleFactor;
      float dy = (midY - prevPinchMid.y) * scaleFactor;
      camTarget.x -= cos(camYaw) * dx;
      camTarget.z += sin(camYaw) * dx;
      camTarget.y += dy;

      updateCartesianFromSpherical();
    }

    prevPinchDist = currDist;
    prevPinchMid.set(midX, midY);
    return;
  }

  if (wasPinching) {
    wasPinching = false;
    prevPinchDist = -1;
    prevTouchX = mouseX;
    prevTouchY = mouseY;
    return;
  }

  prevPinchDist = -1;

  if (draggingSlider) {
    updateSliderFromX(mouseX);
    return;
  }

  if (draggingOrbit) {
    float dx = mouseX - prevTouchX;
    float dy = mouseY - prevTouchY;
    prevTouchX = mouseX;
    prevTouchY = mouseY;

    camYaw -= dx * 0.0065f;
    camPitch += dy * 0.0055f;
    updateCartesianFromSpherical();
  }
}

void mouseReleased() {
  draggingSlider = false;
  draggingOrbit = false;
  wasPinching = false;
  prevPinchDist = -1;
}

void keyPressed() {
  if (key == 'h' || key == 'H') uiVisible = !uiVisible;
  if (key == '1') applyCameraPreset(0);
  if (key == '2') applyCameraPreset(1);
  if (key == '3') applyCameraPreset(2);
  if (key == '4') applyCameraPreset(3);
}

float smoothstep(float edge0, float edge1, float x) {
  float t = constrain((x - edge0) / (edge1 - edge0), 0.0f, 1.0f);
  return t * t * (3.0f - 2.0f * t);
}
