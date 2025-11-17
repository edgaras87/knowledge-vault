---
title: Multi-Slice Example  
date: 2025-11-02
tags: 
    - spring
    - architecture
    - project-layout
    - best-practices
    - cheatsheet
summary: A scalable project layout for Spring Boot applications with 10–30 entities using vertical slices.
aliases:
  - Multi-slice example cheatsheet
---

# Multi-slice example for 10–30 entities

When your service grows beyond a handful of entities, folder salad looms. After all,

Once you pass five entities, a flat “one-entity demo” turns into a Where’s-Waldo. The fix is to group by **feature/bounded context** (vertical slices), and keep the same **edges ↔ core** rhythm inside each slice. For an inventory system, four slices keep you sane:

* **catalog/** — products, categories, brands
* **inventory/** — warehouses, stock levels, movements
* **sales/** — customers, orders, order items
* **procurement/** — suppliers, purchase orders

Here’s a paste-ready skeleton you can drop into `src/`. It scales without devolving into folder salad.

```text
inventory-service/
├─ build.gradle / pom.xml
├─ README.md
├─ .editorconfig
├─ .gitignore
└─ src
   ├─ main
   │  ├─ java/com/edge/inventory
   │  │  ├─ web/                                  # HTTP edge: controllers, DTOs, ProblemDetail
   │  │  │  ├─ catalog/
   │  │  │  │  ├─ ProductController.java
   │  │  │  │  ├─ CategoryController.java
   │  │  │  │  ├─ dto/
   │  │  │  │  │  ├─ CreateProductRequest.java
   │  │  │  │  │  ├─ UpdateProductRequest.java
   │  │  │  │  │  ├─ ProductResponse.java
   │  │  │  │  │  ├─ CreateCategoryRequest.java
   │  │  │  │  │  └─ CategoryResponse.java
   │  │  │  │  └─ mapper/
   │  │  │  │     ├─ ProductWebMapper.java
   │  │  │  │     └─ CategoryWebMapper.java
   │  │  │  ├─ inventory/
   │  │  │  │  ├─ WarehouseController.java
   │  │  │  │  ├─ StockItemController.java       # /warehouses/{id}/stock
   │  │  │  │  ├─ dto/
   │  │  │  │  │  ├─ CreateWarehouseRequest.java
   │  │  │  │  │  ├─ WarehouseResponse.java
   │  │  │  │  │  ├─ AdjustStockRequest.java
   │  │  │  │  │  └─ StockItemResponse.java
   │  │  │  │  └─ mapper/
   │  │  │  │     ├─ WarehouseWebMapper.java
   │  │  │  │     └─ StockWebMapper.java
   │  │  │  ├─ sales/
   │  │  │  │  ├─ OrderController.java
   │  │  │  │  ├─ CustomerController.java
   │  │  │  │  ├─ dto/
   │  │  │  │  │  ├─ CreateOrderRequest.java     # contains line items
   │  │  │  │  │  ├─ OrderResponse.java
   │  │  │  │  │  ├─ CreateCustomerRequest.java
   │  │  │  │  │  └─ CustomerResponse.java
   │  │  │  │  └─ mapper/
   │  │  │  │     ├─ OrderWebMapper.java
   │  │  │  │     └─ CustomerWebMapper.java
   │  │  │  ├─ procurement/
   │  │  │  │  ├─ SupplierController.java
   │  │  │  │  ├─ PurchaseOrderController.java
   │  │  │  │  ├─ dto/
   │  │  │  │  │  ├─ CreateSupplierRequest.java
   │  │  │  │  │  ├─ SupplierResponse.java
   │  │  │  │  │  ├─ CreatePurchaseOrderRequest.java
   │  │  │  │  │  └─ PurchaseOrderResponse.java
   │  │  │  │  └─ mapper/
   │  │  │  │     ├─ SupplierWebMapper.java
   │  │  │  │     └─ PurchaseOrderWebMapper.java
   │  │  │  ├─ error/
   │  │  │  │  ├─ GlobalExceptionHandler.java    # @RestControllerAdvice → ProblemDetail
   │  │  │  │  └─ ProblemTypes.java              # stable type URIs / optional codes
   │  │  │  └─ config/
   │  │  │     └─ WebConfig.java
   │  │  ├─ app/                                  # Use-cases: orchestrate domain + repos, @Transactional
   │  │  │  ├─ catalog/
   │  │  │  │  ├─ CreateProductUseCase.java
   │  │  │  │  ├─ UpdateProductUseCase.java
   │  │  │  │  ├─ ListProductsUseCase.java
   │  │  │  │  ├─ CreateCategoryUseCase.java
   │  │  │  │  └─ CatalogService.java            # implements multiple use-case interfaces
   │  │  │  ├─ inventory/
   │  │  │  │  ├─ CreateWarehouseUseCase.java
   │  │  │  │  ├─ AdjustStockUseCase.java
   │  │  │  │  ├─ TransferStockUseCase.java
   │  │  │  │  └─ InventoryService.java
   │  │  │  ├─ sales/
   │  │  │  │  ├─ CreateOrderUseCase.java
   │  │  │  │  ├─ GetOrderUseCase.java
   │  │  │  │  ├─ CreateCustomerUseCase.java
   │  │  │  │  └─ SalesService.java
   │  │  │  ├─ procurement/
   │  │  │  │  ├─ CreateSupplierUseCase.java
   │  │  │  │  ├─ CreatePurchaseOrderUseCase.java
   │  │  │  │  └─ ProcurementService.java
   │  │  │  └─ exception/
   │  │  │     ├─ ProductNotFoundException.java
   │  │  │     ├─ CategoryNotFoundException.java
   │  │  │     ├─ WarehouseNotFoundException.java
   │  │  │     ├─ InsufficientStockException.java
   │  │  │     ├─ OrderNotFoundException.java
   │  │  │     └─ SupplierNotFoundException.java
   │  │  ├─ domain/                               # Entities + repository ports (interfaces)
   │  │  │  ├─ catalog/
   │  │  │  │  ├─ Product.java
   │  │  │  │  ├─ Category.java
   │  │  │  │  ├─ Brand.java
   │  │  │  │  ├─ ProductRepository.java
   │  │  │  │  └─ CategoryRepository.java
   │  │  │  ├─ inventory/
   │  │  │  │  ├─ Warehouse.java
   │  │  │  │  ├─ StockItem.java                 # product_id + warehouse_id + qty
   │  │  │  │  ├─ StockMovement.java             # audit of adjustments/transfers
   │  │  │  │  ├─ WarehouseRepository.java
   │  │  │  │  └─ StockRepository.java
   │  │  │  ├─ sales/
   │  │  │  │  ├─ Customer.java
   │  │  │  │  ├─ Order.java
   │  │  │  │  ├─ OrderItem.java
   │  │  │  │  ├─ OrderRepository.java
   │  │  │  │  └─ CustomerRepository.java
   │  │  │  ├─ procurement/
   │  │  │  │  ├─ Supplier.java
   │  │  │  │  ├─ PurchaseOrder.java
   │  │  │  │  ├─ PurchaseOrderItem.java
   │  │  │  │  └─ PurchaseOrderRepository.java
   │  │  │  └─ common/
   │  │  │     ├─ BaseEntity.java
   │  │  │     ├─ Money.java                     # value object (amount + currency)
   │  │  │     └─ Quantity.java                  # value object for stock quantities
   │  │  └─ infra/                               # Adapters/config
   │  │     ├─ persistence/
   │  │     │  ├─ JpaConfig.java
   │  │     │  └─ SpecBuilders.java              # optional: Spring Data Specifications
   │  │     ├─ mapping/
   │  │     │  └─ MapStructConfig.java
   │  │     └─ security/
   │  │        └─ SecurityConfig.java            # optional later
   │  └─ resources
   │     ├─ application.yml
   │     ├─ application-local.yml
   │     ├─ db/migration/
   │     │  ├─ V1__init_catalog.sql
   │     │  ├─ V2__init_inventory.sql
   │     │  ├─ V3__init_sales.sql
   │     │  └─ V4__init_procurement.sql
   │     └─ logback-spring.xml
   └─ test
      ├─ java/com/edge/inventory
      │  ├─ web/catalog/
      │  │  ├─ ProductControllerIT.java
      │  │  └─ CategoryControllerIT.java
      │  ├─ web/inventory/
      │  │  └─ WarehouseControllerIT.java
      │  ├─ app/catalog/
      │  │  └─ CatalogServiceTest.java
      │  ├─ app/inventory/
      │  │  └─ InventoryServiceTest.java
      │  ├─ domain/catalog/
      │  │  └─ ProductRepositoryIT.java
      │  └─ domain/inventory/
      │     └─ StockRepositoryIT.java
      └─ resources/
         └─ application-test.yml
```

### Why this stays navigable with 10–30 entities

* **Vertical slices**: You always know where to look. All “product” stuff is under `catalog/product*` across layers.
* **Repeatable pattern**: Each slice has `web/app/domain` subfolders with parallel names. Muscle memory beats search.
* **Shared core**: `domain/common` holds value objects and base types used cross-slice, not a dumping ground.

### Relationships that won’t tangle your brain

* `Product` ↔ `Category` (many-to-one).
* `StockItem` ties `Product` + `Warehouse` with `quantity`.
* `Order` has `OrderItem`s (each references `Product` and a snapshot price).
* `PurchaseOrder` mirrors `Order` but with `Supplier` and incoming quantities.
* Keep aggregates small: `Order` owns `OrderItem`s; `Product` **doesn’t** own `StockItem` rows (inventory slice does).

### MapStruct placement rule (unchanged)

* Web mappers live **at the edge** (`web/.../mapper`), converting `Request ↔ Command` and `Entity ↔ Response`.
* If you add persistence projections/specs, put those mappers in `infra/mapping/`.

```java
// infra/mapping/MapStructConfig.java
@Configuration
@MapperConfig(
  componentModel = "spring",
  unmappedTargetPolicy = ReportingPolicy.ERROR,
  nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public class MapStructConfig {}
```

Then annotate each mapper: `@Mapper(config = MapStructConfig.class)`.

### Use-case seams (how services don’t balloon)

* One service per slice is fine early: `CatalogService`, `InventoryService`, etc., **implement multiple use-case interfaces**.
* When a service grows obese, split by workflow: e.g., `OrderingService` vs `OrderQueryService`.
* Keep **commands/inputs** narrow: `CreateOrderRequest` includes line items; `AdjustStockRequest` is just product+warehouse+delta.

### ProblemDetail catalog (so errors stay boring)

* Keep `GlobalExceptionHandler` in `web/error/`.
* Put stable `type` URIs in `ProblemTypes` like:

  * `/problems/catalog/product-not-found`
  * `/problems/inventory/insufficient-stock`
* Base URI (`problems.base`) in `application.yml`, read as `@ConfigurationProperties` so you don’t hardcode strings.

### Flyway starter pack (one file per slice)

`V1__init_catalog.sql`

```sql
CREATE TABLE category(
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL UNIQUE
);

CREATE TABLE product(
  id BIGSERIAL PRIMARY KEY,
  sku TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  category_id BIGINT NOT NULL REFERENCES category(id)
);
```

`V2__init_inventory.sql`

```sql
CREATE TABLE warehouse(
  id BIGSERIAL PRIMARY KEY,
  code TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL
);

CREATE TABLE stock_item(
  product_id BIGINT NOT NULL REFERENCES product(id),
  warehouse_id BIGINT NOT NULL REFERENCES warehouse(id),
  quantity NUMERIC(19,2) NOT NULL DEFAULT 0,
  PRIMARY KEY(product_id, warehouse_id)
);

CREATE TABLE stock_movement(
  id BIGSERIAL PRIMARY KEY,
  product_id BIGINT NOT NULL REFERENCES product(id),
  warehouse_id BIGINT NOT NULL REFERENCES warehouse(id),
  delta NUMERIC(19,2) NOT NULL,
  reason TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`V3__init_sales.sql`

```sql
CREATE TABLE customer(
  id BIGSERIAL PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL
);

CREATE TABLE "order"(
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customer(id),
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_item(
  order_id BIGINT NOT NULL REFERENCES "order"(id),
  line_no INT NOT NULL,
  product_id BIGINT NOT NULL REFERENCES product(id),
  qty NUMERIC(19,2) NOT NULL,
  unit_price NUMERIC(19,2) NOT NULL,
  PRIMARY KEY(order_id, line_no)
);
```

`V4__init_procurement.sql`

```sql
CREATE TABLE supplier(
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  email TEXT
);

CREATE TABLE purchase_order(
  id BIGSERIAL PRIMARY KEY,
  supplier_id BIGINT NOT NULL REFERENCES supplier(id),
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE purchase_order_item(
  purchase_order_id BIGINT NOT NULL REFERENCES purchase_order(id),
  line_no INT NOT NULL,
  product_id BIGINT NOT NULL REFERENCES product(id),
  qty NUMERIC(19,2) NOT NULL,
  unit_cost NUMERIC(19,2) NOT NULL,
  PRIMARY KEY(purchase_order_id, line_no)
);
```

### Testing layout that mirrors slices

* **Controller IT** per slice (`web/catalog/*IT.java`) asserting `Location`, JSON shape, and ProblemDetails.
* **Service tests** per slice for business rules (stock never negative, order totals).
* **Repo slice tests** (`@DataJpaTest`) for constraints (`UNIQUE`, FKs) and query specs.

### Navigation aids so you never get lost

* Add `package-info.java` in each slice (`web/catalog`, `app/inventory`, …) with a one-paragraph doc of what lives there.
* Put a tiny `README.md` in each top slice folder explaining flows and key use-cases.
* Keep DTO names **scenario-first** (`AdjustStockRequest`, `TransferStockRequest`) rather than generic (`UpdateWarehouse`).

