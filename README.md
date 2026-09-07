/*
=========================================================
                    FARASHOOTERS
=========================================================

 Arduino UNO
 Honor 400 5G Browser Display via USB OTG

 PHYSICAL BUTTONS
 D2 = LEFT
 D3 = RIGHT
 D4 = SHOOT

 BUTTON WIRING
 D2 ---- LEFT BUTTON ---- GND
 D3 ---- RIGHT BUTTON --- GND
 D4 ---- SHOOT BUTTON --- GND

 USB
 Arduino UNO <---- USB OTG ----> Honor 400 5G

 Serial:
 115200 baud

 Browser commands:
 LEFT
 RIGHT
 SHOOT
 MENU_LEFT
 MENU_RIGHT

 Arduino sends game state as JSON.
=========================================================
*/

#define LEFT_BUTTON   2
#define RIGHT_BUTTON  3
#define SHOOT_BUTTON 4

const int SCREEN_W = 320;
const int SCREEN_H = 240;

// =====================================================
// PLAYER
// =====================================================

int playerX = 160;

const int playerY = 210;
const int PLAYER_W = 16;
const int PLAYER_H = 12;

int playerSpeed = 5;

int lives = 3;

// =====================================================
// BULLETS
// =====================================================

#define MAX_BULLETS 3

struct Bullet {
  bool active;
  int x;
  int y;
};

Bullet bullets[MAX_BULLETS];

int maxBullets = 1;
int bulletSpeed = 7;

unsigned long lastShot = 0;
const unsigned long SHOT_DELAY = 180;

// =====================================================
// ENEMIES
// =====================================================

#define MAX_ENEMIES 7

struct Enemy {
  bool active;
  int x;
  int y;
  int speed;
  int health;
};

Enemy enemies[MAX_ENEMIES];

// =====================================================
// GAME
// =====================================================

int stage = 1;
int score = 0;
int enemiesKilled = 0;
int enemiesNeeded = 10;

// =====================================================
// BOSS
// =====================================================

bool bossActive = false;

int bossX = 130;
int bossY = 35;

const int BOSS_W = 60;
const int BOSS_H = 35;

int bossHealth = 40;
int bossMaxHealth = 40;

int bossDirection = 1;
int bossSpeed = 3;

// =====================================================
// GAME STATES
// =====================================================

bool gameOver = false;
bool gameWon = false;
bool stageComplete = false;

// =====================================================
// MENU
// =====================================================

bool inMenu = true;

int menuOption = 0;

const int MENU_OPTIONS = 2;

// =====================================================
// TIMERS
// =====================================================

unsigned long lastEnemySpawn = 0;
unsigned long lastBulletUpdate = 0;
unsigned long lastStateSend = 0;

// =====================================================
// BUTTON STATE
// =====================================================

bool lastShootState = HIGH;

// =====================================================
// CLEAR BULLETS
// =====================================================

void clearBullets() {

  for (int i = 0; i < MAX_BULLETS; i++) {

    bullets[i].active = false;
    bullets[i].x = 0;
    bullets[i].y = 0;

  }
}

// =====================================================
// CLEAR ENEMIES
// =====================================================

void clearEnemies() {

  for (int i = 0; i < MAX_ENEMIES; i++) {

    enemies[i].active = false;
    enemies[i].x = 0;
    enemies[i].y = 0;
    enemies[i].speed = 0;
    enemies[i].health = 0;

  }
}

// =====================================================
// START STAGE
// =====================================================

void startStage() {

  stageComplete = false;
  bossActive = false;

  enemiesKilled = 0;

  playerX = 160;

  bossHealth = bossMaxHealth;
  bossX = 130;
  bossDirection = 1;

  clearBullets();
  clearEnemies();

  lastEnemySpawn = millis();
}

// =====================================================
// RESET GAME
// =====================================================

void resetGame() {

  stage = 1;
  score = 0;
  enemiesKilled = 0;
  lives = 3;

  bossActive = false;
  bossHealth = bossMaxHealth;

  gameOver = false;
  gameWon = false;
  stageComplete = false;

  inMenu = true;
  menuOption = 0;

  playerX = 160;

  clearBullets();
  clearEnemies();
}

// =====================================================
// START GAME
// =====================================================

void startGame() {

  inMenu = false;
  gameOver = false;
  gameWon = false;
  stageComplete = false;

  stage = 1;
  score = 0;
  lives = 3;

  startStage();
}

// =====================================================
// SHOOT
// =====================================================

void shootBullet() {

  if (millis() - lastShot < SHOT_DELAY) {
    return;
  }

  for (int i = 0; i < maxBullets; i++) {

    if (!bullets[i].active) {

      bullets[i].active = true;

      bullets[i].x =
        playerX + PLAYER_W / 2;

      bullets[i].y = playerY;

      lastShot = millis();

      break;
    }
  }
}

// =====================================================
// PHYSICAL BUTTONS
// =====================================================

void handlePhysicalButtons() {

  // LEFT

  if (digitalRead(LEFT_BUTTON) == LOW) {

    if (inMenu) {

      menuOption--;

      if (menuOption < 0) {
        menuOption = MENU_OPTIONS - 1;
      }

      delay(150);

    } else if (!gameOver && !gameWon) {

      playerX -= playerSpeed;

      if (playerX < 10) {
        playerX = 10;
      }
    }
  }

  // RIGHT

  if (digitalRead(RIGHT_BUTTON) == LOW) {

    if (inMenu) {

      menuOption++;

      if (menuOption >= MENU_OPTIONS) {
        menuOption = 0;
      }

      delay(150);

    } else if (!gameOver && !gameWon) {

      playerX += playerSpeed;

      if (playerX >
          SCREEN_W - PLAYER_W - 10) {

        playerX =
          SCREEN_W - PLAYER_W - 10;
      }
    }
  }

  // SHOOT

  bool shootState =
    digitalRead(SHOOT_BUTTON);

  if (shootState == LOW &&
      lastShootState == HIGH) {

    if (inMenu) {

      if (menuOption == 0) {
        startGame();
      }

    } else if (gameOver) {

      resetGame();

    } else if (gameWon) {

      resetGame();

    } else {

      shootBullet();
    }
  }

  lastShootState = shootState;
}

// =====================================================
// SERIAL COMMANDS
// =====================================================

void handleCommand(char *command) {

  if (strcmp(command, "LEFT") == 0) {

    if (!inMenu &&
        !gameOver &&
        !gameWon) {

      playerX -= playerSpeed;

      if (playerX < 10) {
        playerX = 10;
      }
    }
  }

  else if (strcmp(command, "RIGHT") == 0) {

    if (!inMenu &&
        !gameOver &&
        !gameWon) {

      playerX += playerSpeed;

      if (playerX >
          SCREEN_W - PLAYER_W - 10) {

        playerX =
          SCREEN_W - PLAYER_W - 10;
      }
    }
  }

  else if (strcmp(command, "SHOOT") == 0) {

    if (inMenu) {

      if (menuOption == 0) {
        startGame();
      }

    } else if (gameOver) {

      resetGame();

    } else if (gameWon) {

      resetGame();

    } else {

      shootBullet();
    }
  }

  else if (strcmp(command, "MENU_LEFT") == 0) {

    if (inMenu) {

      menuOption--;

      if (menuOption < 0) {
        menuOption = MENU_OPTIONS - 1;
      }
    }
  }

  else if (strcmp(command, "MENU_RIGHT") == 0) {

    if (inMenu) {

      menuOption++;

      if (menuOption >= MENU_OPTIONS) {
        menuOption = 0;
      }
    }
  }
}

// =====================================================
// READ SERIAL
// =====================================================

char commandBuffer[32];
byte commandIndex = 0;

void readSerialCommands() {

  while (Serial.available()) {

    char c = Serial.read();

    if (c == '\n' || c == '\r') {

      if (commandIndex > 0) {

        commandBuffer[commandIndex] = '\0';

        handleCommand(commandBuffer);

        commandIndex = 0;
      }

    } else {

      if (commandIndex <
          sizeof(commandBuffer) - 1) {

        commandBuffer[commandIndex++] = c;
      }
    }
  }
}

// =====================================================
// UPDATE BULLETS
// =====================================================

void updateBullets() {

  if (millis() - lastBulletUpdate < 25) {
    return;
  }

  lastBulletUpdate = millis();

  for (int i = 0; i < MAX_BULLETS; i++) {

    if (!bullets[i].active) {
      continue;
    }

    bullets[i].y -= bulletSpeed;

    if (bullets[i].y < 0) {
      bullets[i].active = false;
    }
  }
}

// =====================================================
// SPAWN ENEMIES
// =====================================================

void spawnEnemies() {

  if (enemiesKilled >= enemiesNeeded) {

    bossActive = true;

    return;
  }

  if (millis() -
      lastEnemySpawn < 700) {

    return;
  }

  for (int i = 0;
       i < MAX_ENEMIES;
       i++) {

    if (!enemies[i].active) {

      enemies[i].active = true;

      enemies[i].x =
        random(10, SCREEN_W - 25);

      enemies[i].y = 25;

      enemies[i].speed =
        random(1, 3);

      enemies[i].health = 1;

      lastEnemySpawn = millis();

      break;
    }
  }
}

// =====================================================
// UPDATE ENEMIES
// =====================================================

void updateEnemies() {

  for (int i = 0;
       i < MAX_ENEMIES;
       i++) {

    if (!enemies[i].active) {
      continue;
    }

    enemies[i].y +=
      enemies[i].speed;

    if (enemies[i].y > SCREEN_H) {

      enemies[i].active = false;

      lives--;

      if (lives <= 0) {
        gameOver = true;
      }
    }
  }
}

// =====================================================
// UPDATE BOSS
// =====================================================

void updateBoss() {

  bossX +=
    bossDirection *
    bossSpeed;

  if (bossX <= 10) {
    bossDirection = 1;
  }

  if (bossX >=
      SCREEN_W - BOSS_W - 10) {

    bossDirection = -1;
  }

  if (bossHealth <= 0) {

    bossActive = false;
    gameWon = true;
  }
}

// =====================================================
// COLLISIONS
// =====================================================

void checkCollisions() {

  // BULLET VS ENEMY

  for (int b = 0;
       b < MAX_BULLETS;
       b++) {

    if (!bullets[b].active) {
      continue;
    }

    for (int e = 0;
         e < MAX_ENEMIES;
         e++) {

      if (!enemies[e].active) {
        continue;
      }

      if (
        bullets[b].x >= enemies[e].x &&
        bullets[b].x <= enemies[e].x + 20 &&
        bullets[b].y >= enemies[e].y &&
        bullets[b].y <= enemies[e].y + 15
      ) {

        bullets[b].active = false;
        enemies[e].active = false;

        score += 10;
        enemiesKilled++;

        break;
      }
    }
  }

  // BULLET VS BOSS

  if (bossActive) {

    for (int b = 0;
         b < MAX_BULLETS;
         b++) {

      if (!bullets[b].active) {
        continue;
      }

      if (
        bullets[b].x >= bossX &&
        bullets[b].x <= bossX + BOSS_W &&
        bullets[b].y >= bossY &&
        bullets[b].y <= bossY + BOSS_H
      ) {

        bullets[b].active = false;

        bossHealth--;

        score += 5;
      }
    }
  }
}

// =====================================================
// UPDATE GAME
// =====================================================

void updateGame() {

  if (inMenu ||
      gameOver ||
      gameWon) {

    return;
  }

  updateBullets();

  if (!bossActive) {

    spawnEnemies();
    updateEnemies();

  } else {

    updateBoss();
  }

  checkCollisions();

  if (bossActive &&
      bossHealth <= 0) {

    bossActive = false;
    gameWon = true;
  }
}

// =====================================================
// SEND JSON GAME STATE
// =====================================================

void sendGameState() {

  if (millis() -
      lastStateSend < 50) {

    return;
  }

  lastStateSend = millis();

  Serial.print("{");

  Serial.print("\"menu\":");
  Serial.print(inMenu ? "true" : "false");

  Serial.print(",\"gameOver\":");
  Serial.print(gameOver ? "true" : "false");

  Serial.print(",\"gameWon\":");
  Serial.print(gameWon ? "true" : "false");

  Serial.print(",\"menuOption\":");
  Serial.print(menuOption);

  Serial.print(",\"playerX\":");
  Serial.print(playerX);

  Serial.print(",\"playerY\":");
  Serial.print(playerY);

  Serial.print(",\"score\":");
  Serial.print(score);

  Serial.print(",\"lives\":");
  Serial.print(lives);

  Serial.print(",\"stage\":");
  Serial.print(stage);

  Serial.print(",\"killed\":");
  Serial.print(enemiesKilled);

  Serial.print(",\"needed\":");
  Serial.print(enemiesNeeded);

  Serial.print(",\"bossActive\":");
  Serial.print(bossActive ? "true" : "false");

  Serial.print(",\"bossX\":");
  Serial.print(bossX);

  Serial.print(",\"bossY\":");
  Serial.print(bossY);

  Serial.print(",\"bossHealth\":");
  Serial.print(bossHealth);

  Serial.print(",\"bossMaxHealth\":");
  Serial.print(bossMaxHealth);

  // ENEMIES

  Serial.print(",\"enemies\":[");

  bool firstEnemy = true;

  for (int i = 0;
       i < MAX_ENEMIES;
       i++) {

    if (!enemies[i].active) {
      continue;
    }

    if (!firstEnemy) {
      Serial.print(",");
    }

    firstEnemy = false;

    Serial.print("{\"x\":");
    Serial.print(enemies[i].x);

    Serial.print(",\"y\":");
    Serial.print(enemies[i].y);

    Serial.print("}");
  }

  Serial.print("]");

  // BULLETS

  Serial.print(",\"bullets\":[");

  bool firstBullet = true;

  for (int i = 0;
       i < MAX_BULLETS;
       i++) {

    if (!bullets[i].active) {
      continue;
    }

    if (!firstBullet) {
      Serial.print(",");
    }

    firstBullet = false;

    Serial.print("{\"x\":");
    Serial.print(bullets[i].x);

    Serial.print(",\"y\":");
    Serial.print(bullets[i].y);

    Serial.print("}");
  }

  Serial.print("]");

  Serial.println("}");
}

// =====================================================
// SETUP
// =====================================================

void setup() {

  Serial.begin(115200);

  pinMode(
    LEFT_BUTTON,
    INPUT_PULLUP
  );

  pinMode(
    RIGHT_BUTTON,
    INPUT_PULLUP
  );

  pinMode(
    SHOOT_BUTTON,
    INPUT_PULLUP
  );

  randomSeed(
    analogRead(A0)
  );

  clearBullets();
  clearEnemies();

  delay(1000);

  Serial.println(
    "{\"connection\":\"FARASHOOTERS_READY\"}"
  );
}

// =====================================================
// LOOP
// =====================================================

void loop() {

  readSerialCommands();

  handlePhysicalButtons();

  updateGame();

  sendGameState();
}
