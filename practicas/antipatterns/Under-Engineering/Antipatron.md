#INSTITUTO TECNOLOGICO DE TIJUANA
##Antonio de Jesus Galvan Godinez

## ¿Qué es "Under-Engineering" (Infra-ingeniería)?

El **Under-Engineering** es la práctica de no aplicar suficiente rigor, diseño o esfuerzo de ingeniería en el desarrollo de una solución. Es lo opuesto al "Over-Engineering" (sobre-ingeniería), pero ambos son considerados malas prácticas.

Un sistema con infra-ingeniería suele manifestarse como:

* **Frágil**: Falla inesperadamente ante casos borde o un ligero aumento de carga.
* **Incompleto**: No cumple con todos los requisitos funcionales o no funcionales (como rendimiento, accesibilidad o seguridad).
* **Difícil de Mantener**: El código es "demasiado simple", a menudo escrito sin pensar en la escalabilidad, reutilización o legibilidad, lo que genera deuda técnica inmediata.
* **Costoso a Largo Plazo**: Requiere refactorizaciones constantes o parches ("hotfixes") para solucionar problemas que un diseño adecuado habría prevenido.

Mientras que el *Over-Engineering* resuelve problemas futuros que quizás nunca existan, el *Under-Engineering* falla en resolver adecuadamente los problemas presentes. Nuestro objetivo es encontrar el balance correcto, creando soluciones robustas y mantenibles sin añadir complejidad innecesaria.

## Descripción

Este PR introduce // 1. Se crea una interfaz genérica
public interface IDiscountStrategy
{
    decimal ApplyDiscount(decimal total);
    // ¿Qué pasa si un futuro descuento necesita el 'Customer' o 'Order'?
    // La abstracción ya es incorrecta.
}

// 2. Se crea una clase para un solo caso de uso
public class NewCustomerDiscountStrategy : IDiscountStrategy
{
    public decimal ApplyDiscount(decimal total)
    {
        return total * 0.90m; // 10% de descuento
    }
}

// 3. Se crea una clase para el caso "ninguno"
public class NoDiscountStrategy : IDiscountStrategy
{
    public decimal ApplyDiscount(decimal total)
    {
        return total;
    }
}

// 4. Se crea un "Factory" para gestionar las estrategias
public class DiscountStrategyFactory
{
    public IDiscountStrategy GetStrategy(Customer customer)
    {
        if (customer.IsNew)
        {
            return new NewCustomerDiscountStrategy();
        }

        return new NoDiscountStrategy();
    }
}

// 5. El Servicio principal ahora es más complejo y tiene más dependencias
public class OrderProcessingService
{
    private readonly DiscountStrategyFactory _strategyFactory;

    // Se añade una dependencia que debe ser inyectada y mockeada
    public OrderProcessingService(DiscountStrategyFactory strategyFactory)
    {
        _strategyFactory = strategyFactory;
    }

    public decimal CalculateTotal(Order order, Customer customer)
    {
        decimal total = order.Subtotal;
        
        // Toda esta indirección solo para hacer un 'if'
        IDiscountStrategy strategy = _strategyFactory.GetStrategy(customer);
        total = strategy.ApplyDiscount(total);

        return total;
    }
}

## Enfoque de Ingeniería: Evitando el "Over-Engineering"

En este desarrollo, se ha priorizado un enfoque pragmático para evitar el **"Over-Engineering" (sobre-ingeniería)**, adhiriéndonos a principios clave como:

* **YAGNI (You Aren't Gonna Need It)**: No se ha implementado funcionalidad que no sea estrictamente necesaria para los requisitos actuales. Se evita así añadir código para "posibles casos futuros" que podrían no llegar nunca.
* **KISS (Keep It Simple, Stupid)**: La solución se ha mantenido lo más simple posible. Se ha preferido una implementación clara y directa sobre patrones de diseño complejos que no aportaban un valor tangible en este momento.

### Justificación

El objetivo es entregar valor de forma rápida y mantener una base de código limpia y fácil de entender. Si bien la solución es simple, no cae en el **"Under-Engineering" (infra-ingeniería)**, ya que:

* Cumple con todos los requisitos funcionales y no funcionales definidos.
* Incluye las pruebas necesarias para garantizar su robustez.
* Sigue los estándares de código y buenas prácticas del proyecto.

Este enfoque nos permite iterar más rápido y añadir complejidad solo cuando sea realmente necesaria y esté justificada por nuevos requisitos.

## 3. Consecuencias del "Over-Engineering"

* **Mantenimiento:**
    * **Mayor Carga Cognitiva:** Un nuevo desarrollador ahora debe entender 5 archivos/clases (`IDiscountStrategy`, 2 implementaciones, 1 Factory, 1 Servicio) en lugar de leer un solo método.
    * **Rigidez:** Paradójicamente, la abstracción prematura puede ser rígida. Si un nuevo descuento (ej. "Cupón de 20€") no encaja con la firma `ApplyDiscount(decimal total)`, la interfaz `IDiscountStrategy` debe romperse, violando el Principio Abierto/Cerrado.
    * **Complejidad en Pruebas:** Se requiere *mockear* el `DiscountStrategyFactory` para probar el servicio, añadiendo complejidad innecesaria a las pruebas unitarias.

* **Rendimiento:**
    * Aunque mínimo en este caso, se introduce una sobrecarga (overhead) por la creación de múltiples objetos (`Factory`, `Strategy`) e indirección (llamadas a métodos de interfaz) donde no era necesario.

* **Escalabilidad:**
    * No hay un beneficio real de escalabilidad porque la abstracción se basó en una suposición. Se gastó tiempo en esto en lugar de enfocarse en optimizaciones reales (ej. consultas de base de datos, caché).

---

## 4. Solución Correctiva

**Buenas Prácticas (KISS y YAGNI):**
Implementar la solución más simple que funcione y cumpla *exactamente* con el requisito actual.

**Patrones Alternativos (Refactorización Iterativa):**
Comenzar simple y aplicar la "Regla del Tres": no abstraer hasta que se tengan al menos tres casos de uso concretos.

**Código Corregido (El balance correcto):**
La lógica se implementa directamente en el servicio. Es legible, fácil de probar y cumple el requisito.

```csharp
public class OrderProcessingService
{
    private const decimal NEW_CUSTOMER_DISCOUNT_PERCENTAGE = 0.10m; // 10%

    // No se necesitan dependencias extra
    public OrderProcessingService() { }

    public decimal CalculateTotal(Order order, Customer customer)
    {
        if (order == null || customer == null)
        {
            throw new ArgumentNullException("La orden y el cliente no pueden ser nulos.");
        }

        decimal total = order.Subtotal;

        // Lógica simple, clara y directa
        if (customer.IsNew)
        {
            decimal discountAmount = total * NEW_CUSTOMER_DISCOUNT_PERCENTAGE;
            total -= discountAmount;
        }

        // Si en el FUTURO se añade un descuento VIP:
        // else if (customer.IsVip)
        // {
        //     total -= total * VIP_DISCOUNT;
        // }
        //
        // ...Solo cuando esto se vuelva inmanejable (3+ casos),
        // se refactoriza al Patrón Strategy.

        return total;
    }
}

// Pruebas Unitarias Simples (No requieren Mocks)
[Fact]
public void CalculateTotal_NewCustomer_Applies10PercentDiscount()
{
    // Arrange
    var service = new OrderProcessingService(); // Sin dependencias
    var newCustomer = new Customer { IsNew = true };
    var order = new Order { Subtotal = 100.00m };
    var expectedTotal = 90.00m;

    // Act
    var actualTotal = service.CalculateTotal(order, newCustomer);

    // Assert
    Assert.Equal(expectedTotal, actualTotal);
}



###Puntos Clave (Síntesis)
Claridad y Lenguaje Técnico: Usar los términos: "Antipatrón", "Over-Engineering", "KISS", "YAGNI", "Abstracción Prematura", "Patrón Strategy", "Carga Cognitiva" y "Refactorización Iterativa".

Mensaje Clave: El Over-Engineering no te hace un mejor ingeniero; te hace un ingeniero más lento. La verdadera habilidad de ingeniería senior es saber cuándo no usar un patrón de diseño complejo.

Conclusión: No resuelvas problemas que no tienes. Resuelve el problema actual de forma limpia y mantén la flexibilidad para refactorizar en el futuro, cuando la complejidad real lo justifique.
