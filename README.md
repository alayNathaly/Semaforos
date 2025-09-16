import java.util.Random;
import java.util.concurrent.Semaphore;

public class Semaforos {

    static class Datos {
        Semaphore semSalida = new Semaphore(0, true);

        // Barrera por vuelta
        Semaphore[] semVuelta = new Semaphore[] {
                new Semaphore(0, true),
                new Semaphore(0, true),
                new Semaphore(0, true)
        };

        int[] terminadosVuelta = new int[]{0, 0, 0};
        long[][] tiemposVuelta = new long[3][3]; // [auto][vuelta]
        String[] nombres = new String[]{"Auto Rojo", "Auto Azul", "Auto Verde"};

        String[] podio = new String[3];
        int lugar = 1;

        synchronized void registrarLlegada(String nombre) {
            if (lugar <= 3) {
                podio[lugar - 1] = nombre;
                lugar++;
            }
        }

        int calcularGanadorDeVuelta(int v) {
            long mejor = Long.MAX_VALUE;
            int idx = 0;
            for (int a = 0; a < 3; a++) {
                if (tiemposVuelta[a][v] < mejor) {
                    mejor = tiemposVuelta[a][v];
                    idx = a;
                }
            }
            return idx;
        }
    }

    static class Auto implements Runnable {
        int idx;
        String nombre;
        Datos datos;
        Random rnd = new Random();

        Auto(int idx, Datos datos) {
            this.idx = idx;
            this.nombre = datos.nombres[idx];
            this.datos = datos;
        }

        @Override
        public void run() {
            // Primero avisamos que está listo (NO corre todavía)
            System.out.println(nombre + " listo para arrancar");

            try {
                datos.semSalida.acquire(); // espera la señal de salida
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            for (int v = 0; v < 3; v++) {
                long inicio = System.currentTimeMillis();

                try {
                    Thread.sleep(300 + rnd.nextInt(500));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }

                long fin = System.currentTimeMillis();
                long dur = fin - inicio;
                datos.tiemposVuelta[idx][v] = dur;

                boolean soyUltimo = false;
                synchronized (datos) {
                    datos.terminadosVuelta[v]++;
                    if (datos.terminadosVuelta[v] == 3) {
                        int lider = datos.calcularGanadorDeVuelta(v);
                        System.out.println("Vuelta " + (v + 1) + ": va adelante " + datos.nombres[lider]);
                        datos.semVuelta[v].release(3);
                        soyUltimo = true;
                    }
                }

                if (!soyUltimo) {
                    try {
                        datos.semVuelta[v].acquire();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            }

            datos.registrarLlegada(nombre);
        }
    }

    static class Juez implements Runnable {
        Datos datos;
        Juez(Datos d) { this.datos = d; }

        @Override
        public void run() {
            try {
                // Pequeña pausa para dejar que los autos impriman "listo"
                Thread.sleep(300);

                System.out.println("\n=== COMIENZA LA CARRERA ===");
                System.out.println("Juez: 3");
                Thread.sleep(700);
                System.out.println("Juez: 2");
                Thread.sleep(700);
                System.out.println("Juez: 1");
                Thread.sleep(700);
                System.out.println("Juez: ¡YA!");
                datos.semSalida.release(3);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Datos datos = new Datos();

        Thread a0 = new Thread(new Auto(0, datos));
        Thread a1 = new Thread(new Auto(1, datos));
        Thread a2 = new Thread(new Auto(2, datos));
        Thread juez = new Thread(new Juez(datos));

        // Primero arrancamos los autos para que impriman "listo"
        a0.start();
        a1.start();
        a2.start();

        // Pequeña pausa para asegurar que los 3 autos ya imprimieron "listo"
        Thread.sleep(200);

        // Arrancamos al juez después
        juez.start();

        a0.join();
        a1.join();
        a2.join();
        juez.join();

        System.out.println("\n=== PUESTOS FINALES ===");
        System.out.println("1er lugar: " + datos.podio[0]);
        System.out.println("2do lugar: " + datos.podio[1]);
        System.out.println("3er lugar: " + datos.podio[2]);
    }
}
