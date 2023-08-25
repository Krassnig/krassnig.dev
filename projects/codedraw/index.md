# CodeDraw

CodeDraw is a beginner-friendly drawing library which can be used to create pictures, animations and even interactive applications.
It is designed for people who are just starting to learn programming, enabling them to create graphical applications.

The source code, documentation and additional examples can be found in the [CodeDraw Repository](https://github.com/Krassnig/CodeDraw).

If you are interested in the detailed designed decision that shaped this library, consider reading my bachelor thesis on
[CodeDraw: A Graphics Library for Novice Programmers](./CodeDraw%20Thesis.pdf).

The following two programs are implemented using CodeDraw and porovide examples of how this library can be used.

## Particle Gravity

The first program is a particle game where each particle moves towards the mouse cursor.
The particles' speed increases as they get closer to the cursor.

As this example demonstates, CodeDraw programs can be implemented using structured programming alone.
Users of the library do not have to know how to implement classes or functions.

<video controls autoplay loop src="./particle-gravity-video.mp4"></video>

```java
import codedraw.*;

public class ParticleGravity {
	public static void main(String[] args) {
		CodeDraw cd = new CodeDraw();
		EventScanner es = cd.getEventScanner();

		final int particleCount = 100000;
		final double exponent = -0.25 * Math.sqrt(2) - 0.5;
		final double speed = 100;

		// game state
		int mouseX = cd.getWidth() / 2;
		int mouseY = cd.getHeight() / 2;
		double[] particlesX = new double[particleCount];
		double[] particlesY = new double[particleCount];

		for (int i = 0; i < particleCount; i++) {
			particlesX[i] = Math.random() * cd.getWidth();
			particlesY[i] = Math.random() * cd.getHeight();
		}

		// game loop
		while (!cd.isClosed()) {
			// event handling
			while (es.hasEventNow()) {
				if (es.hasMouseMoveEvent()) {
					MouseMoveEvent a = es.nextMouseMoveEvent();
					mouseX = a.getX();
					mouseY = a.getY();
				}
				else {
					es.nextEvent();
				}
			}

			// game logic
			for (int i = 0; i < particleCount; i++) {
				double distanceX = mouseX - particlesX[i];
				double distanceY = mouseY - particlesY[i];

				double movementFactor = speed * Math.pow(distanceX * distanceX + distanceY * distanceY, exponent);

				particlesX[i] += distanceX * movementFactor;
				particlesY[i] += distanceY * movementFactor;
			}

			// rendering
			cd.clear(Palette.BLACK);
			for (int i = 0; i < particleCount; i++) {
				cd.setPixel((int)particlesX[i], (int)particlesY[i], Palette.RED);
			}
			cd.show(16);
		}
	}
}
```

## Game of Life

The second example is an implementation of Conway's Game of Life.
The rules that determine the state of each cell (white or black) can be found in the
[Conway's Game of Life Wikipedia article](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life).

This implementation also allows the user place black cells by clicking on a white cell and dragging the mouse,
as well as place white cells by clicking on a black cell and dragging the mouse.

<video controls autoplay loop src="./game-of-life-video.mp4"></video>

```java
import codedraw.*;

public class GameOfLife {
	private static final int FIELD_SIZE = 10;

	public static void main(String[] args) {
		final int size = 1 << 6; // must be power of 2
		final int mask = size - 1;

		CodeDraw cd = new CodeDraw(size * FIELD_SIZE, size * FIELD_SIZE);
		EventScanner es = cd.getEventScanner();

		boolean[][] field = createRandomBooleans(size);

		boolean isMouseDown = false;
		boolean setValue = false;

		for (int i = 0; !cd.isClosed(); i++) {
			while (es.hasEventNow()) {
				if (es.hasMouseDownEvent()) {
					MouseDownEvent a = es.nextMouseDownEvent();
					isMouseDown = true;
					// the value that is used to draw cells depends on the state
					// of the cell where the initial click happens
					// if that cell was white then every subsequent mouse move will draw black cells.
					int x = a.getX() / FIELD_SIZE;
					int y = a.getY() / FIELD_SIZE;
					setValue = field[x][y] = !field[x][y];
				}
				else if (es.hasMouseUpEvent() || es.hasMouseLeaveEvent()) {
					es.nextEvent();
					isMouseDown = false;
				}
				else if (es.hasMouseMoveEvent()) {
					MouseMoveEvent a = es.nextMouseMoveEvent();
					if (isMouseDown) {
						field[a.getX() / FIELD_SIZE][a.getY() / FIELD_SIZE] = setValue;
					}
				}
				else {
					es.nextEvent();
				}
			}

			// update to next generation only every eighth render
			if (i % 8 == 0) {
				field = simulateNextGeneration(field, mask);
			}

			render(cd, field);
		}
	}

	private static boolean[][] createRandomBooleans(int size) {
		boolean[][] field = new boolean[size][size];

		for (int x = 0; x < size; x++) {
			for (int y = 0; y < size; y++) {
				field[x][y] = Math.random() > 0.5;
			}
		}

		return field;
	}

	public static boolean[][] simulateNextGeneration(boolean[][] field, int mask) {
		final int radius = 1;
		boolean[][] nextField = new boolean[field.length][field.length];

		for (int x = 0; x < field.length; x++) {
			for (int y = 0; y < field[x].length; y++) {
				int sum = 0;

				for (int xn = -radius; xn <= radius; xn++) {
					for (int yn = -radius; yn <= radius; yn++) {
						// ignore the center    and only sum if the neighbor is alive
						if ((xn != 0 || yn != 0) && field[(x + xn) & mask][(y + yn) & mask]) {
							sum++;
						}
					}
				}

				if (field[x][y]) {
					nextField[x][y] = sum == 2 || sum == 3;
				}
				else {
					nextField[x][y] = sum == 3;
				}
			}
		}

		return nextField;
	}

	public static void render(CodeDraw cd, boolean[][] field) {
		for (int x = 0; x < field.length; x++) {
			for (int y = 0; y < field[x].length; y++) {
				cd.setColor(field[x][y] ? Palette.BLACK : Palette.WHITE);
				cd.fillSquare(FIELD_SIZE * x, FIELD_SIZE * y, FIELD_SIZE);
			}
		}

		cd.show(16);
	}
}
```