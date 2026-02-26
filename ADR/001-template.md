# ADR-001: Refactorización del módulo de autenticación por violaciones críticas de seguridad y Clean Code
## Contexto
El sistema actual es un servicio Spring Boot que expone puntos finales para el registro y el inicio de sesión. La lógica de autenticación se implementa en `AuthService`, el acceso a datos en `UserRepository`, y el controlador `AuthController` recibe parámetros con nombres crípticos. La base de datos PostgreSQL se accede mediante SQL construido mediante concatenación de cadenas y las contraseñas se almacenan mediante MD5. Durante la auditoría de la fase 2 y las pruebas de la fase 3 detectamos múltiples problemas: inyección SQL, hash débil de contraseñas, exposición del hash en la respuesta, credenciales hardcodeadas, recursos no cerrados, atributos públicos y nombres no descriptivos (ver tabla de hallazgos en `ADR/02-fase/tabla-de-hallazgos.md`). Estas fallas representan un riesgo alto para los usuarios — contraseñas comprometidas, datos filtrados y potenciales accesos no autorizados — y afectan directamente al equipo de desarrollo que debe soportar un código difícil de mantener. El negocio también se ve comprometido por posibles vulnerabilidades explotables en producción.

## Decisión
1. Se refactorizará el acceso a la base de datos para usar `PreparedStatement` y parametrización, eliminando la concatenación de SQL y mitigando inyección. Esto aplica el principio de mínima exposición y SRP al delegar construcción de consultas a una clase de utilidades o a JPA.
2. Se reemplazará el hashing MD5 por un esquema seguro (bcrypt/Argon2) con un servicio dedicado y se eliminará la devolución de hashes en las respuestas. De este modo se cumple el principio de OCP al poder cambiar el algoritmo sin modificar la lógica de negocio.
3. Se encapsularán los campos de `User` con getters/setters y se validará existencia de usuario en el registro para evitar duplicados. También se realizarán validaciones de entrada en el controlador y se mejorarán los nombres de parámetros y variables (`username`, `password`, `email`). Estos cambios mejorarán la legibilidad (Clean Code) y respetan SRP y DIP al abstraer dependencias.
4. Las credenciales de la base de datos se moverán a variables de entorno o al fichero `application.properties`, eliminando hardcode y facilitando configuración segura en distintos entornos.
5. Se añadirá cierre automático de recursos (try-with-resources) y se inyectará un `DataSource` gestionado por Spring en lugar de crear conexiones manualmente, promoviendo reuse y bajo acoplamiento.

La arquitectura del módulo pasará de un conjunto de clases con lógica mezclada a una estructura en la que:
- `AuthController` gestiona validaciones y traducción de peticiones.
- `AuthService` orquesta la lógica de autenticación, delegando hashing y validaciones a componentes separados.
- `UserRepository` usa JDBC templating o JPA con consultas parametrizadas.
- Se introduce un `PasswordHasher`/`CryptoService` para abstracción del algoritmo.

## Consecuencias
- **Ganas**: mayor seguridad contra inyección y robo de credenciales, código más mantenible, más fácil de testear y extensible, cumplimiento de principios SOLID y buenas prácticas de Clean Code, menor riesgo de filtración de datos, y configuración más flexible.
- **Pierdes/Riesgos**: el refactorizado consume tiempo de desarrollo y pruebas; puede introducir regresiones si no se cubre con tests; la adopción de bibliotecas (bcrypt) añade una dependencia y requiere migración de hashes existentes.

## Alternativas consideradas
- Reescribir el módulo desde cero: descartado porque implicaría mayor esfuerzo y riesgos de integración, además de que la refactorización incremental es suficiente.
- Corregir solo la seguridad (usar PreparedStatement y cambiar el hash) sin tocar nombres ni estructura: descartado porque dejaría el código difícil de leer y mantener, perpetuando deuda técnica.
- Mantener la implementación actual y agregar un firewall/IDS externo: descartado por no solucionar los problemas fundamentales en el código y porque la vulnerabilidad seguiría existiendo internamente.
