# Árbol de navegación — GB97 Order

Mapa de pantallas de la app organizado por punto de entrada y por rol de usuario.
Última actualización: 2026-07-01 (tras eliminar el rol `guest`, reemplazado por
el rol `customer` de Marketplace).

## Flujo previo al login

```
ModeSelectionScreen (primera instalación)
├─ "Panel de Clientes" → CustomerHomeScreen (rol Cliente/Marketplace)
└─ "Panel de Vendedores/Admin" → LoginScreen
        ├─ Selección de cuenta (si hay varias) → AccountSelectionScreen
        ├─ "Olvidé mi contraseña" → ForgotPassScreen → VerificationCodeScreen → PassRecoverySuccess
        ├─ "Registrarme" → RegisterScreen1 → RegisterScreen2
        └─ Login exitoso → HomeScreen (rol según el usuario)

UpdateScreen (si hay actualización pendiente) → Tienda de apps
```

## Rol Admin / Super-admin / Vendedor

```
HomeScreen
├─ 1. Inicio — HomeWelcomeScreen
│   ├─ Avatar → MyProfileScreen
│   ├─ Selector/creación de Sucursal → BranchDetailsManagementScreen
│   ├─ Dashboard de ventas → SalesReportsScreen
│   └─ Accesos rápidos: Grupos · Combos · Extras · Catálogos · Reportes
│
├─ 2. Productos — ItemsScreen
│   ├─ Detalle de ítem → ItemDetailsScreen (variantes, grupos, precios)
│   ├─ Crear/editar ítem → ItemDetailsManagementScreen
│   └─ Importar desde Excel · Exportar · Ver eliminados
│
├─ 3. Clientes — ClientsScreen
│   ├─ Detalle de cliente → ClientDetailsScreen
│   │   ├─ Sucursales del cliente → ClientBranchDetailsScreen
│   │   └─ Documentos (PDF/PPT) → DocumentDetailsScreen
│   ├─ Crear/editar cliente → ClientDetailsManagementScreen
│   └─ Importar desde Excel · Archivar/eliminar en lote
│
├─ 4. Proformas — QuotationsScreen
│   ├─ Detalle → QuotationDetailsScreen (convertir a pedido, enviar, PDF)
│   └─ Nueva proforma → NewOrderScreen1 (cliente → ítems → revisión)
│
├─ 5. Pedidos — OrdersScreen
│   ├─ Detalle → OrderDetailsScreen
│   │   ├─ Seguimiento de entrega → DeliveryDetailsScreen → mapa en vivo / ruta
│   │   ├─ Pagos → Deuna / PayPal / Payphone / Cuotas
│   │   └─ Ver/generar factura → InvoiceDetailsScreen
│   └─ Nuevo pedido → NewOrderScreen1
│
└─ 6. Facturas — InvoicesScreen
    ├─ Pestañas: "Mis Facturas" / "Importadas"
    ├─ Detalle → InvoiceDetailsScreen (PDF, enviar, anular)
    ├─ Nueva factura → NewInvoiceScreen1
    ├─ Configuración SRI → InvoicesConfigScreen
    └─ Validación de comprobantes → ReceiptsCheckScreen

Mi Perfil (MyProfileScreen) — accesible desde el avatar en cualquier pestaña
├─ Mis Datos → ProfileConfigScreen
├─ Mi Comercio → OrganizationScreen
│   ├─ Información de la empresa → MyOrganizationScreen
│   ├─ Gestión de Personal → StaffManagementScreen → UserDetailsManagementScreen (roles, permisos, sucursal)
│   └─ Reportes de ventas → SalesReportsScreen
├─ Planes y Suscripciones → SubscriptionsScreen
├─ Preferencias → UserPreferencesScreen
├─ Tutoriales → TutorialsScreen → reproductor de video
├─ Permisos de la app, Políticas de privacidad, Cuentas vinculadas (Google/Apple)
└─ Cerrar sesión
```

## Rol Cliente (Marketplace / B2C)

```
CustomerHomeScreen
├─ Productos — CustomerItemsScreen → CustomerItemDetailScreen → "Agregar al carrito"
├─ Catálogos — CustomerCatalogsScreen → CatalogViewScreen
├─ Carrito — CustomerCartScreen
│   └─ Checkout → CustomerPersonalInfoScreen (si no autenticado) o pago directo (autenticado)
└─ Perfil — CustomerProfileScreen
    ├─ [No autenticado] Iniciar sesión / Registrarse / Cambiar modo
    └─ [Autenticado] Mis Datos · Mis Proformas (CustomerQuotationsScreen) · Mis Pedidos (CustomerOrdersScreen) · Preferencias · Cerrar sesión
```

## Rol Transportista (Carrier)

```
HomeScreen
├─ Entregas — OrdersScreen (filtrado a pedidos asignados)
│   └─ Detalle → DeliveryDetailsScreen → mapa en vivo, ruta, estado de entrega
└─ Mi Perfil — versión reducida
```

---

## Notas

- El alcance funcional cambia por **rol**
  (`admin`/`super-admin`/`seller`/`customer`/`carrier`), definido en
  `lib/src/ui/home/home_screen.dart` y en la barra inferior
  `lib/src/widgets/navigation/custom_bottom_navigation_bar.dart` — ese es el
  mejor punto de partida en el código para entender "qué ve cada tipo de
  usuario".
- El rol **`guest` (invitado) fue eliminado del código** en 2026-07-01: estaba
  reemplazado hace tiempo por el rol `customer` de Marketplace y ya no era
  alcanzable desde ninguna pantalla viva.
- El hub administrativo real (sucursales, personal, suscripción, reportes)
  cuelga de **Mi Perfil → Mi Comercio**, no de la pestaña "Inicio".
