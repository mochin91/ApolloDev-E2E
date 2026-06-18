# Filament 4 — Reference Guide

## Installation

```bash
composer require filament/filament:"^4.0"
php artisan filament:install --panels
```

## Panel Provider

```php
// app/Providers/Filament/AdminPanelProvider.php
class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->id('admin')
            ->path('admin')
            ->login()
            ->colors(['primary' => Color::Amber])
            ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
            ->discoverPages(in: app_path('Filament/Pages'), for: 'App\\Filament\\Pages')
            ->discoverWidgets(in: app_path('Filament/Widgets'), for: 'App\\Filament\\Widgets')
            ->middleware([
                EncryptCookies::class,
                AddQueuedCookiesToResponse::class,
                StartSession::class,
                AuthenticateSession::class,
                ShareErrorsFromSession::class,
                VerifyCsrfToken::class,
                SubstituteBindings::class,
                DisableBladeIconComponents::class,
                DispatchServingFilamentEvent::class,
            ])
            ->authMiddleware([Authenticate::class]);
    }
}
```

## Resource Structure

```bash
php artisan make:filament-resource Product --generate
```

```
app/Filament/Resources/
└── ProductResource.php
    └── Pages/
        ├── CreateProduct.php
        ├── EditProduct.php
        └── ListProducts.php
```

```php
class ProductResource extends Resource
{
    protected static ?string $model = Product::class;
    protected static ?string $navigationIcon = 'heroicon-o-rectangle-stack';
    protected static ?string $navigationGroup = 'Catalog';
    protected static ?int $navigationSort = 1;

    public static function form(Form $form): Form
    {
        return $form->schema([...]);
    }

    public static function table(Table $table): Table
    {
        return $table->columns([...])->filters([...])->actions([...])->bulkActions([...]);
    }

    public static function getRelations(): array { return []; }
    public static function getPages(): array
    {
        return [
            'index'  => Pages\ListProducts::route('/'),
            'create' => Pages\CreateProduct::route('/create'),
            'edit'   => Pages\EditProduct::route('/{record}/edit'),
        ];
    }
}
```

## Form Components

```php
use Filament\Forms\Components\{TextInput, Select, Textarea, Toggle, DatePicker, FileUpload, Repeater, Section, Grid};

// Text
TextInput::make('name')->required()->maxLength(255)->columnSpanFull()
TextInput::make('price')->numeric()->prefix('Rp')->minValue(0)

// Select — BelongsTo
Select::make('category_id')
    ->relationship('category', 'name')
    ->searchable()
    ->preload()
    ->required()

// Select — static options
Select::make('status')
    ->options(['active' => 'Active', 'inactive' => 'Inactive'])
    ->default('active')

// Multi-select — BelongsToMany
Select::make('tags')
    ->relationship('tags', 'name')
    ->multiple()
    ->preload()

// Textarea
Textarea::make('description')->rows(4)->columnSpanFull()

// Toggle / Checkbox
Toggle::make('is_active')->default(true)

// Date
DatePicker::make('published_at')->displayFormat('d/m/Y')

// File Upload
FileUpload::make('thumbnail')
    ->image()
    ->directory('products')
    ->disk('public')

// Repeater
Repeater::make('items')
    ->relationship()
    ->schema([
        TextInput::make('name')->required(),
        TextInput::make('quantity')->numeric()->required(),
    ])
    ->columns(2)

// Layout
Section::make('Details')->schema([...])->columns(2)->collapsed()
Grid::make(2)->schema([...])
```

## Table Columns & Filters

```php
use Filament\Tables\Columns\{TextColumn, BooleanColumn, ImageColumn, BadgeColumn};
use Filament\Tables\Filters\{SelectFilter, TernaryFilter, Filter};

// Columns
TextColumn::make('name')->searchable()->sortable()
TextColumn::make('category.name')->label('Category')->sortable()
TextColumn::make('price')->money('IDR')->sortable()
TextColumn::make('created_at')->dateTime('d/m/Y H:i')->sortable()->toggleable(isToggledHiddenByDefault: true)
BooleanColumn::make('is_active')->label('Active')
BadgeColumn::make('status')
    ->colors(['success' => 'active', 'danger' => 'inactive', 'warning' => 'pending'])

// Filters
SelectFilter::make('status')->options([...])
SelectFilter::make('category')->relationship('category', 'name')
TernaryFilter::make('is_active')
Filter::make('created_at')
    ->form([DatePicker::make('from'), DatePicker::make('until')])
    ->query(fn ($query, array $data) =>
        $query->when($data['from'], fn ($q) => $q->whereDate('created_at', '>=', $data['from']))
              ->when($data['until'], fn ($q) => $q->whereDate('created_at', '<=', $data['until']))
    )
```

## Actions

```php
use Filament\Tables\Actions\{EditAction, DeleteAction, ViewAction, BulkActionGroup, DeleteBulkAction};
use Filament\Actions\{CreateAction, Action};

// Table row actions
->actions([
    EditAction::make(),
    DeleteAction::make()->requiresConfirmation(),
    Action::make('approve')
        ->icon('heroicon-o-check')
        ->color('success')
        ->requiresConfirmation()
        ->action(fn (Model $record) => $record->approve()),
])

// Bulk actions
->bulkActions([
    BulkActionGroup::make([DeleteBulkAction::make()]),
])

// Header actions (on list page)
protected function getHeaderActions(): array
{
    return [CreateAction::make()];
}
```

## Widgets

```php
// Stats Overview
class StatsOverviewWidget extends BaseWidget
{
    protected function getStats(): array
    {
        return [
            Stat::make('Total Products', Product::count())
                ->description('All time')
                ->color('success'),
            Stat::make('Revenue', 'Rp ' . number_format(Order::sum('total')))
                ->chart([7, 3, 4, 5, 6, 3, 5]),
        ];
    }
}

// Chart Widget
class RevenueChart extends ChartWidget
{
    protected static ?string $heading = 'Revenue';

    protected function getData(): array
    {
        return [
            'datasets' => [['label' => 'Revenue', 'data' => [...]]],
            'labels' => ['Jan', 'Feb', ...],
        ];
    }

    protected function getType(): string { return 'line'; }
}
```

## Authorization

```php
// On Resource — policy-based
public static function canCreate(): bool { return auth()->user()->can('create', static::$model); }
public static function canEdit(Model $record): bool { return auth()->user()->can('update', $record); }
public static function canDelete(Model $record): bool { return auth()->user()->can('delete', $record); }
public static function canViewAny(): bool { return auth()->user()->can('viewAny', static::$model); }
```

## RelationManagers

```bash
php artisan make:filament-relation-manager ProductResource OrderItems order_item_id
```

```php
class OrderItemsRelationManager extends RelationManager
{
    protected static string $relationship = 'orderItems';

    public function form(Form $form): Form { return $form->schema([...]); }
    public function table(Table $table): Table { return $table->columns([...]); }
}

// On Resource
public static function getRelations(): array
{
    return [OrderItemsRelationManager::class];
}
```

## Custom Pages

```bash
php artisan make:filament-page Settings
```

```php
class Settings extends Page
{
    protected static ?string $navigationIcon = 'heroicon-o-cog';
    protected static string $view = 'filament.pages.settings';
}
```

## Anti-Patterns

- ❌ `->options(Model::all()->pluck('name', 'id'))` — runs on every render → use `->relationship()` or lazy closure
- ❌ Missing `->preload()` on searchable selects — causes blank dropdown on mobile
- ❌ `canAccess()` not overridden — all authenticated users see all resources
- ❌ No `->columnSpanFull()` on long fields — breaks grid layout
- ❌ `Repeater` without `->relationship()` — data not persisted via Eloquent
