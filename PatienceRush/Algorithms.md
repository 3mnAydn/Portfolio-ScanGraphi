```markdown
# Core Mechanics & Algorithms

This document highlights the core architectural decisions and custom algorithms built for PatienceRush using Java and the Android SDK. To protect proprietary business logic, only foundational structural snippets (such as the custom physics engine and map generation logic) are shared below.

---

### 1. Physics Engine and Collision Management (`Character` Class)

These methods define the core physics rules of the game. The `update` function handles gravity, speed calculations, and updates the character's position when riding moving blocks. Collision checks are performed sequentially—first on the Y-axis (`checkCollisionY`) and then on the X-axis (`checkCollisionX`)—ensuring the character reacts correctly to obstacles without clipping through walls.

```java
public void update() {
    speed[1] += gConst;
    setY(getY() + (float) speed[1]);
    checkCollisionY();

    setX(getX() + (float) speed[0]);
    checkCollisionX();

    if (attachedBlock != null) {
        setX(getX() + attachedBlock.getCurrentDx());
        setY(getY() + attachedBlock.getCurrentDy());
    }
}

private void checkCollisionY() {
    attachedBlock = null;
    rectChar.set(getX(), getY(), getX() + getWidth(), getY() + getHeight());

    for (Block b : blocks) {
        rectBlock.set(b.getX(), b.getY(), b.getX() + b.getWidth(), b.getY() + b.getHeight());
        if (!RectF.intersects(rectChar, rectBlock)) continue;

        float overlapY = Math.min(rectChar.bottom, rectBlock.bottom)
                       - Math.max(rectChar.top,    rectBlock.top);
        if (overlapY <= GameConstants.SNAG_TOLERANCE) continue;

        if (speed[1] > 0 && rectChar.bottom > rectBlock.top) {
            setY(rectBlock.top - getHeight());
            speed[1] = 0;
            jumpReady = true;
            attachedBlock = b;
        } else if (speed[1] < 0 && rectChar.top < rectBlock.bottom) {
            setY(rectBlock.bottom);
            speed[1] = 0;
        }
    }
}

private void checkCollisionX() {
    rectChar.set(getX(), getY(), getX() + getWidth(), getY() + getHeight());

    for (Block b : blocks) {
        rectBlock.set(b.getX(), b.getY(), b.getX() + b.getWidth(), b.getY() + b.getHeight());
        if (!RectF.intersects(rectChar, rectBlock)) continue;

        float overlapX = Math.min(rectChar.right,  rectBlock.right)
                       - Math.max(rectChar.left,   rectBlock.left);
        if (overlapX <= GameConstants.SNAG_TOLERANCE) continue;

        if (speed[0] > 0 && rectChar.right > rectBlock.left) {
            setX(rectBlock.left - getWidth());
            speed[0] = 0;
        } else if (speed[0] < 0 && rectChar.left < rectBlock.right) {
            setX(rectBlock.right);
            speed[0] = 0;
        }
    }
}

```

### 2. Moving Block Logic (`Block` Class)

The `update` method in the `Block` class enables obstacles to move back and forth within defined boundaries on either the horizontal or vertical axis. It also calculates the delta position required to smoothly carry the character along when they land on the block.

```java
public void update() {
    if (moveType == 0) return;

    float prevX = getX();
    float prevY = getY();

    if (moveType == 1) {
        setX(getX() + moveSpeed * moveDirection);
        if (Math.abs(getX() - originalX) >= moveRange) moveDirection *= -1;
    } else if (moveType == 2) {
        setY(getY() + moveSpeed * moveDirection);
        if (Math.abs(getY() - originalY) >= moveRange) moveDirection *= -1;
    }

    setCurrentDx(getX() - prevX);
    setCurrentDy(getY() - prevY);
}

```

### 3. Procedural Map Generation

The `generateHardcoreMap` function dynamically constructs level layouts. It determines the number of blocks, their dimensions, structural types, and movement probabilities by combining randomization with mathematical multipliers based on the current level difficulty. The level goal is placed statically at the end of the generated path.

```java
private void generateHardcoreMap(Context context, ArrayList<Block> blocks,
        int lvl, int screenW, int screenH, float devSx, float devSy) {

    Random rand = new Random();
    double diffRatio = Math.min(lvl / GameConstants.DIFFICULTY_DIVISOR,
                                GameConstants.MAX_DIFFICULTY_RATIO)
                     * GameConstants.DIFFICULTY_SCALE_FACTOR;

    int blockCount = (int) Math.min(
        GameConstants.MAX_BLOCKS,
        Math.max(GameConstants.MIN_BLOCKS,
            GameConstants.BLOCK_COUNT_BASE * Math.pow(lvl, GameConstants.BLOCK_COUNT_POWER))
    );

    float curX = startBlockX + GameConstants.GAP_SIZE * devSx;
    float curY = startBlockY;

    float movingChance = Math.min(
        lvl * GameConstants.MOVING_BLOCK_CHANCE_MULTIPLIER,
        GameConstants.MAX_MOVING_BLOCK_CHANCE
    );

    for (int i = 0; i < blockCount; i++) {
        double typeRoll = rand.nextDouble();
        float bw, bh;

        bw = (GameConstants.BLOCK_WIDTH_BASE
            + rand.nextInt(GameConstants.BLOCK_WIDTH_RANDOM_MAX)
            + GameConstants.BLOCK_WIDTH_BONUS_MULTIPLIER * (1 - (float) diffRatio)) * devSx;
        bh = (GameConstants.BLOCK_HEIGHT_BASE
            + rand.nextInt(GameConstants.BLOCK_HEIGHT_RANDOM_MAX)) * devSy;

        float minY = GameConstants.MIN_Y_POSITION * devSy;
        float maxY = (GameConstants.DESIGN_HEIGHT - GameConstants.MAX_Y_MARGIN) * devSy;
        curY = minY + rand.nextFloat() * (maxY - minY);

        Block block;
        if (typeRoll < GameConstants.TYPE_ROLL_CEILING) {
            block = new GroundBlock(curX, curY - bh * GameConstants.CEILING_HEIGHT_MULTIPLIER, bw, bh, context);
        } else if (typeRoll < GameConstants.TYPE_ROLL_WALL) {
            bw = (GameConstants.WALL_WIDTH_BASE + rand.nextInt(GameConstants.WALL_WIDTH_RANDOM_MAX)) * devSx;
            bh = (GameConstants.WALL_HEIGHT_BASE + rand.nextInt(GameConstants.WALL_HEIGHT_RANDOM_MAX)) * devSy;
            block = new GroundBlock(curX, curY - bh * GameConstants.WALL_Y_OFFSET, bw, bh, context);
        } else {
            block = new GroundBlock(curX, curY, bw, bh, context);
        }

        if (rand.nextFloat() < movingChance) {
            float speed = (GameConstants.BLOCK_WIDTH_BASE * (float) diffRatio) * devSx;
            float range = bw * (GameConstants.RANGE_RATIO_MIN
                        + rand.nextFloat() * (GameConstants.RANGE_RATIO_MAX - GameConstants.RANGE_RATIO_MIN));
            if (rand.nextBoolean()) {
                block.setMovement(1, speed * GameConstants.HORIZONTAL_SPEED_MULTIPLIER, range);
            } else {
                block.setMovement(2, speed * GameConstants.VERTICAL_SPEED_MULTIPLIER,
                                     range * GameConstants.VERTICAL_RANGE_MULTIPLIER);
            }
        }

        blocks.add(block);
        curX += bw + GameConstants.GAP_SIZE * devSx;
    }
    
    blocks.add(new FinishBlock(
        curX + GameConstants.FINISH_BLOCK_OFFSET_X * devSx,
        GameConstants.FINISH_BLOCK_Y * devSy,
        GameConstants.FINISH_BLOCK_WIDTH * devSx,
        screenH, context
    ));
}

```

### 4. Camera Management and Rendering System

These functions govern the screen scaling and the main drawing loop. `calculateViewport` computes the scaling factor against a reference resolution. The `draw` function optimizes rendering performance via Frustum Culling, identifying objects outside the camera bounds and excluding them from the drawing cycle.

```java
private void calculateViewport() {
    scaleFactor = Math.min(
        (float) screenWidth  / GameConstants.REF_WIDTH,
        (float) screenHeight / GameConstants.REF_HEIGHT
    );
}

private void draw() {
    Canvas canvas = surfaceHolder.lockCanvas();
    if (canvas == null) return;
    canvas.drawColor(Color.BLACK);

    float camX = ch.getX() - GameConstants.DESIGN_WIDTH / GameConstants.CAMERA_X_OFFSET;
    float camY = ch.getY() - GameConstants.DESIGN_HEIGHT * GameConstants.CAMERA_Y_OFFSET;

    canvas.save();
    canvas.scale(scaleFactor, scaleFactor);
    canvas.translate(-camX, -camY);

    for (Block b : blocks) {
        if (b.getX() - camX < -GameConstants.CULL_MARGIN) continue;
        if (b.getX() - camX > GameConstants.DESIGN_WIDTH + GameConstants.CULL_MARGIN) continue;
        b.insertWall(canvas);
    }
    ch.insertChar(canvas);
    canvas.restore();

    drawTriangle(canvas, jumpBtnX,  jumpBtnY,  btnRadius, "up",    upTrianglePath);
    drawTriangle(canvas, leftBtnX,  leftBtnY,  btnRadius, "left",  leftTrianglePath);
    drawTriangle(canvas, rightBtnX, rightBtnY, btnRadius, "right", rightTrianglePath);

    surfaceHolder.unlockCanvasAndPost(canvas);
}

```

### 5. Multi-Touch Input Control

The `onTouchEvent` and `checkButtonPress` functions process user interactions with the virtual interface. They are fully designed to support multi-touch scenarios, allowing the player to jump and move simultaneously without input blocking.

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    int action = event.getActionMasked();
    int pointerCount = event.getPointerCount();

    switch (action) {
        case MotionEvent.ACTION_DOWN:
        case MotionEvent.ACTION_POINTER_DOWN: {
            int idx = event.getActionIndex();
            checkButtonPress(event.getX(idx), event.getY(idx), true);
            break;
        }
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_POINTER_UP: {
            int idx = event.getActionIndex();
            checkButtonPress(event.getX(idx), event.getY(idx), false);
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            for (int i = 0; i < pointerCount; i++) {
                checkButtonPress(event.getX(i), event.getY(i), true);
            }
            break;
        }
    }
    return true;
}

private void checkButtonPress(float touchX, float touchY, boolean isDown) {
    if (distance(touchX, touchY, jumpBtnX, jumpBtnY)
            < btnRadius * GameConstants.TOUCH_DISTANCE_MULTIPLIER_JUMP) {
        if (isDown) ch.jump();
    }
    if (distance(touchX, touchY, leftBtnX, leftBtnY)
            < btnRadius * GameConstants.TOUCH_DISTANCE_MULTIPLIER_MOVE) {
        ch.setMovingLeft(isDown);
    }
    if (distance(touchX, touchY, rightBtnX, rightBtnY)
            < btnRadius * GameConstants.TOUCH_DISTANCE_MULTIPLIER_MOVE) {
        ch.setMovingRight(isDown);
    }
}

private double distance(float x1, float y1, float x2, float y2) {
    return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
}

```

```

```
