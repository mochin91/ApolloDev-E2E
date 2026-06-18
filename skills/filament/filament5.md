# Filament 5 — Reference Guide & Migration from v4

## Installation

```bash
composer require filament/filament:"^5.0"
php artisan filament:install --panels
php artisan filament:upgrade  # from v4
```

## Breaking Changes from Filament 4

### 1. Namespace Changes
```php
// v4
use Filament\Tables\Columns\BadgeColumn;

// v5 — BadgeColumn removed, use TextColumn with badge()
use Filament\Tables\Columns\TextColumn;
TextColumn::make('status')->badge()->color(fn ($state) => match($state) {
    'active' => 'success',
    'inactive' => 'danger',
});
```

### 2. Panel Configuration — new `->spa()` mode
```php
// v5 — SPA mode available (Livewire navigate)
$panel->spa()
```

### 3. Form Field — `->live()` replaces `->reactive()`
```php
// v4
TextInput::make('name')->reactive()

// v5
TextInput::make('name')->live()
TextInput::make('name')->live(onBlur: true)  // only on blur
```

### 4. `->afterStateUpdated()` signature change
```php
// v4
->afterStateUpdated(function ($state, callable $set) { $set('slug', str($state)->slug()); })

// v5
->afterStateUpdated(function ($state, Set $set) { $set('slug', str($state)->slug()); })
// Must import: use Filament\Forms\Set;
```

### 5. `->hidden()` / `->visible()` with closures
```php
// v4
->hidden(fn ($get) => $get('type') !== 'special')

// v5 — same syntax, but $get now typed
->hidden(fn (Get $get) => $get('type') !== 'special')
// Must import: use Filament\Forms\Get;
```

### 6. Table `->query()` on List pages
```php
// v4 — override getTableQuery()
protected function getTableQuery(): Builder { return Product::query()->withTrashed(); }

// v5 — override getEloquentQuery()
public static function getEloquentQuery(): Builder { return parent::getEloquentQuery()->withTrashed(); }
```

### 7. Notifications — new static syntax
```php
// v4
$this->notify('success', 'Saved!');

// v5
use Filament\Notifications\Notification;
Notification::make()->title('Saved!')->success()->send();
```

### 8. Action `->form()` closures
```php
// v5 — Actions can have inline forms
Action::make('reject')
    ->form([
        Textarea::make('reason')->required(),
    ])
    ->action(function (array $data, Model $record): void {
        $record->reject($data['reason']);
    })
```

### 9. `InfoList` — new read-only view (replaces custom view pages)
```php
use Filament\Infolists\Infolist;
use Filament\Infolists\Components\{TextEntry, Section};

public static function infolist(Infolist $infolist): Infolist
{
    return $infolist->schema([
        Section::make('Details')->schema([
            TextEntry::make('name'),
            TextEntry::make('status')->badge(),
        ])->columns(2),
    ]);
}

// Add view page to Resource
'view' => Pages\ViewProduct::route('/{record}'),
```

### 10. Cluster — new grouping for Resources & Pages
```php
// v5 only — group related resources
php artisan make:filament-cluster Inventory

class Inventory extends Cluster
{
    protected static ?string $navigationIcon = 'heroicon-o-squares-2x2';
}

// On Resource
class ProductResource extends Resource
{
    protected static ?string $cluster = Inventory::class;
}
```

## Full Panel Provider (v5)

```php
class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->id('admin')
            ->path('admin')
            ->spa()                    // v5: SPA mode
            ->login()
            ->colors(['primary' => Color::Indigo])
            ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
            ->discoverPages(in: app_path('Filament/Pages'), for: 'App\\Filament\\Pages')
            ->discoverWidgets(in: app_path('Filament/Widgets'), for: 'App\\Filament\\Widgets')
            ->discoverClusters(in: app_path('Filament/Clusters'), for: 'App\\Filament\\Clusters')
            ->middleware([...])
            ->authMiddleware([Authenticate::class])
            ->plugins([
                SpatieLaravelPermissionsPlugin::make(),   // example
            ]);
    }
}
```

## Form Components (v5 additions)

```php
// ToggleButtons — replaces some Radio patterns
ToggleButtons::make('status')
    ->options(['draft' => 'Draft', 'published' => 'Published'])
    ->colors(['draft' => 'warning', 'published' => 'success'])
    ->icons(['draft' => 'heroicon-o-pencil', 'published' => 'heroicon-o-check'])
    ->inline()

// ColorPicker
ColorPicker::make('color')->rgba()

// KeyValue
KeyValue::make('metadata')

// Builder — flexible repeater with named blocks
Builder::make('content')
    ->blocks([
        Builder\Block::make('heading')->schema([TextInput::make('text'), Select::make('level')]),
        Builder\Block::make('paragraph')->schema([Textarea::make('content')]),
        Builder\Block::make('image')->schema([FileUpload::make('url')]),
    ])
```

## Table (v5 additions)

```php
// Summarizers
TextColumn::make('price')
    ->money('IDR')
    ->summarize([
        Tables\Columns\Summarizers\Sum::make()->money('IDR'),
        Tables\Columns\Summarizers\Average::make()->money('IDR'),
    ])

// Grouping
->groups([
    Tables\Grouping\Group::make('status')->label('By Status')->collapsible(),
])
->defaultGroup('status')

// Reorder
->reorderable('sort_order')

// Column toggle persistence
->persistColumnSearchesInSession()
->persistFiltersInSession()
```

## Spatie Permissions Integration (v5)

```php
// In panel provider
->plugins([SpatieLaravelPermissionsPlugin::make()])

// On Resource
public static function canViewAny(): bool { return auth()->user()->can('view_any_product'); }
public static function canCreate(): bool { return auth()->user()->can('create_product'); }
public static function canEdit(Model $record): bool { return auth()->user()->can('update_product'); }
public static function canDelete(Model $record): bool { return auth()->user()->can('delete_product'); }
```

## Migration Checklist: v4 → v5

- [ ] Replace `BadgeColumn` → `TextColumn->badge()`
- [ ] Replace `->reactive()` → `->live()`
- [ ] Add `use Filament\Forms\Set;` and `use Filament\Forms\Get;` for closures
- [ ] Replace `$this->notify()` → `Notification::make()->...->send()`
- [ ] Replace `getTableQuery()` → `getEloquentQuery()` on Resources
- [ ] Run `php artisan filament:upgrade` — patches most auto-fixable changes
- [ ] Check all custom Livewire components for wire:navigate compatibility if using `->spa()`
- [ ] Update `composer.json`: `"filament/filament": "^5.0"`
- [ ] Clear caches: `php artisan optimize:clear && php artisan view:clear`

## Anti-Patterns (v5-specific)

- ❌ Using `->reactive()` — deprecated, causes silent failures in v5
- ❌ `BadgeColumn` import — class removed in v5
- ❌ `$this->notify()` — removed, breaks silently
- ❌ Not running `filament:upgrade` — leaves v4 stubs that error at runtime
- ❌ Missing `->spa()` + `wire:navigate` on anchor tags — SPA mode breaks standard `<a href>` links
- ❌ Using `getTableQuery()` on resource — must be `getEloquentQuery()` in v5
