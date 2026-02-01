# BUDDYBOSS APP DEVELOPMENT REFERENCE

## DOCUMENT METADATA
- **Version**: 1.0
- **Last Updated**: January 2026
- **Minimum Plugin Version**: BuddyBoss App Plugin v1.0.4
- **Source**: https://buddyboss.com/resources/dev-docs/app-development/

---

## TABLE OF CONTENTS

```
1. ARCHITECTURE OVERVIEW
2. PLUGIN EXTENSIONS (PHP/WordPress)
   2.1 API_ENDPOINTS
   2.2 API_CACHING
   2.3 API_EXTENDING
   2.4 PUSH_NOTIFICATIONS_AUTO
   2.5 PUSH_NOTIFICATIONS_MANUAL
   2.6 IN_APP_PURCHASES
   2.7 ACCESS_CONTROLS
   2.8 TAB_BAR_SCREENS
   2.9 DEEP_LINKING_PLUGIN
   2.10 GUTENBERG_BLOCKS_PLUGIN
   2.11 WEB_FALLBACKS
   2.12 PROFILE_GROUP_TABS
3. APP EXTENSIONS (React Native)
   3.1 BEST_PRACTICES
   3.2 SCREEN_MODIFICATION
   3.3 SCREEN_CREATION
   3.4 HOOKS_SYSTEM
   3.5 API_FETCHING
   3.6 DATA_STORAGE
   3.7 PUSH_NOTIFICATIONS_APP
   3.8 DEEP_LINKING_APP
   3.9 GUTENBERG_BLOCKS_APP
   3.10 NATIVE_LIBRARIES
4. CLASS_REFERENCE
5. HOOK_REFERENCE
6. FILTER_REFERENCE
7. INTEGRATION_NAMES
8. COMMON_PATTERNS
```

---

## 1. ARCHITECTURE OVERVIEW

```
┌─────────────────────────────────────────────────────────────┐
│                    BUDDYBOSS APP STACK                       │
├─────────────────────────────────────────────────────────────┤
│  MOBILE APP (React Native)                                   │
│  ├── custom_code/index.js (entry point)                     │
│  ├── externalCodeSetup APIs                                  │
│  └── Navigation: Root → Auth/noAuth → Main/All              │
├─────────────────────────────────────────────────────────────┤
│  REST API LAYER                                              │
│  ├── WordPress REST API (/wp-json/wp/v2/)                   │
│  ├── BuddyBoss Platform API (/wp-json/buddyboss/v1/)        │
│  └── BuddyBoss App API (/wp-json/buddyboss-app/)            │
├─────────────────────────────────────────────────────────────┤
│  WORDPRESS PLUGIN (PHP)                                      │
│  ├── BuddyBoss App Plugin                                   │
│  ├── BuddyBoss Platform Plugin                              │
│  └── Custom Plugin Extensions                                │
└─────────────────────────────────────────────────────────────┘
```

**Navigation Stack Structure:**
```
Root (SwitchNavigator)
├── Auth (StackNavigator) - Login, Signup, ForgotPassword, etc.
└── noAuth (StackNavigator)
    ├── Main (BottomTabNavigator) - Tab Bar screens
    └── All (StackNavigator) - All other screens
```

---

## 2. PLUGIN EXTENSIONS (PHP/WordPress)

### 2.1 API_ENDPOINTS

**Purpose**: Create custom REST API endpoints for new functionality.

**When to Use**:
- Custom post types not auto-exposed by WordPress
- Data stored in custom database tables
- Custom business logic not fitting existing APIs

**Implementation**:
```php
// Use WordPress register_rest_route()
// Reference: https://developer.wordpress.org/rest-api/extending-the-rest-api/adding-custom-endpoints/
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/writing-api-endpoints-for-new-functionality/

---

### 2.2 API_CACHING

**Purpose**: Add custom endpoints to BuddyBoss API caching system for performance.

**Base Class**: `BuddyBoss\Performance\Integration\Integration_Abstract`

**Required Methods**:
```php
public function set_up() {
    // Register integration
    $this->register('integration_name');

    // Register setting (appears in BuddyBoss App > Settings > API Caching)
    add_filter('bbapp_admin_performance_custom_setting', [$this, 'register_setting']);

    // Cache endpoints
    $this->cache_endpoint(
        'wp/v2/endpoint',           // endpoint path
        Cache::instance()->month_in_seconds * 60,  // expiration
        ['unique_id' => 'id'],      // args
        true                         // deep_cache (for lists)
    );
}
```

**Cache Purging Methods**:
```php
// Purge list endpoint cache
Cache::instance()->purge_by_group('integration_name');

// Purge single item cache
Cache::instance()->purge_by_group('integration_name_' . $item_id);

// Purge all integration cache
Cache::instance()->purge_by_component('integration_name');
```

**Deep Cache**: For directory/list endpoints, caches each item separately. Only updated items are re-fetched.

**Performance Tip**: Load caching class at MU (must-use) plugin level for fastest response.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/extending-api-caching-to-support-new-api-endpoints/

---

### 2.3 API_EXTENDING

**Purpose**: Add custom data to existing WordPress/BuddyBoss endpoint responses.

**Method 1 - Register REST Field**:
```php
register_rest_field(
    'bp_activity',              // object type
    'custom_field_name',        // attribute name
    [
        'get_callback'    => [$this, 'get_custom_data'],
        'update_callback' => [$this, 'update_custom_data'],
        'schema'          => null,
    ]
);
```

**Method 2 - Response Filters**:
```php
// BuddyBoss Platform: bp_rest_{component_name}
add_filter('bp_rest_activity_prepare_value', [$this, 'modify_response'], 10, 3);

// LearnDash: bbapp_ld_{object_name}
add_filter('bbapp_ld_course_prepare_value', [$this, 'modify_response'], 10, 3);

// WordPress core
add_filter('rest_prepare_post', [$this, 'modify_response'], 10, 3);
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/extending-the-buddyboss-app-api/

---

### 2.4 PUSH_NOTIFICATIONS_AUTO

**Purpose**: Create automated push notifications triggered by events.

**Base Class**: `BuddyBossApp\Notification\IntegrationAbstract`

**Required Methods**:
```php
public function load() {
    // Register push group
    $this->register_push_group('group_name', __('Group Label', 'text-domain'));

    // Register push type
    $this->register_push_type(
        'type_name',
        __('Admin Label', 'text-domain'),
        __('User Label', 'text-domain'),
        ['push_group' => 'group_name']
    );

    // Hook to events
    add_action('save_post_book', [$this, 'send_notification'], 999, 2);
}

public function format_notification($component_name, $component_action, $item_id, $secondary_item_id, $notification_id) {
    // Format web notification
    return [
        'text' => 'Notification text',
        'link' => get_permalink($item_id),
    ];
}
```

**Sending Notifications**:
```php
$this->send_push([
    'primary_text'   => 'Title',
    'secondary_text' => 'Body',
    'user_ids'       => [1, 2, 3],
    'data'           => ['link' => $url],
    'subscription_type' => 'type_name',
    // Optional: extend to web
    'normal_notification' => true,
    'normal_notification_data' => [
        'component_name'    => 'book',
        'component_action'  => 'published',
        'item_id'           => $book_id,
        'secondary_item_id' => $title,
    ],
]);
```

**Batch Processing for Large Groups**:
```php
$jobs = BuddyBossApp\Jobs::instance();
$jobs->add('job_name', ['paged' => 1, 'item_id' => $id]);
$jobs->start();

// Handle job
add_action('bbapp_queue_task_job_name', [$this, 'process_batch']);
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/creating-automatic-push-notifications/

---

### 2.5 PUSH_NOTIFICATIONS_MANUAL

**Purpose**: Add user segment filters for manual push notification targeting.

**Base Class**: `BuddyBossApp\UserSegment\SegmentsAbstract`

**Required Methods**:
```php
public function __construct() {
    // Register group
    $this->add_group('group_name', __('Group Label', 'text-domain'));

    // Register filter
    $this->add_filter('group_name', 'filter_name', ['field_name'], [
        'label' => __('Filter Label', 'text-domain'),
    ]);

    // Register field
    $this->add_field('field_name', 'Checkbox', [
        'options'       => $options_array,
        'multiple'      => true,
        'empty_message' => __('No items found.', 'text-domain'),
    ]);

    $this->load();
}

public function filter_users($user_ids) {
    $filter = $this->get_filter_data_value('filter');
    $selected = (array) $this->get_filter_data_value('field_name');

    // Return filtered user IDs
    return array_merge($user_ids, $filtered_ids);
}

public function render_script() {
    // Optional: custom JavaScript
}
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/extending-manual-push-notifications/

---

### 2.6 IN_APP_PURCHASES

**Purpose**: Support third-party plugins for In-App Purchases (Apple/Google).

**Base Class**: `BuddyBossApp\InAppPurchases\IntegrationAbstract`

**Registration**:
```php
public function register() {
    $this->integration_slug  = 'my-integration';
    $this->integration_type  = 'my-integration';
    $this->integration_label = __('My Integration', 'text-domain');
    $this->item_label        = __('Item', 'text-domain');

    bbapp_iap()->integrations[$this->integration_slug] = [
        'type'    => $this->integration_type,
        'label'   => $this->integration_label,
        'enabled' => true,
        'class'   => self::class,
    ];

    parent::set_up($this->integration_type, $this->integration_label);
    bbapp_iap()->integration[$this->integration_type] = $this::instance();
}
```

**Required Abstract Methods**:
| Method | Purpose | Return |
|--------|---------|--------|
| `iap_linking_options($results)` | Get items for dropdown | `[['id' => '', 'text' => '']]` |
| `iap_integration_ids($results, $ids)` | Convert IDs to labels | `[['id' => '', 'text' => '']]` |
| `item_id_permalink($link, $item_id)` | Get item edit link | URL string |
| `is_purchase_available($bool, $item_id, $integration_item_id)` | Check purchasability | boolean |
| `has_access($item_ids, $order)` | Check user access | boolean |
| `on_order_completed($item_ids, $order)` | Grant access | void |
| `on_order_activate($item_ids, $order)` | Reactivate access | void |
| `on_order_cancelled($item_ids, $order)` | Revoke access | void |
| `on_order_expired($item_ids, $order)` | Expire access | void |

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/extending-the-in-app-purchases-component/

---

### 2.7 ACCESS_CONTROLS

**Purpose**: Create custom conditions for Access Control content restrictions.

**Base Class**: `BuddyBossApp\AccessControls\Integration_Abstract`

**Implementation**:
```php
public function setup() {
    $this->register_condition([
        'condition'              => 'my-condition',
        'items_callback'         => [$this, 'items_callback'],
        'item_callback'          => [$this, 'item_callback'],
        'users_callback'         => [$this, 'users_callback'],
        'labels'                 => [
            'condition_name'          => __('My Condition', 'text-domain'),
            'item_singular'           => __('Item', 'text-domain'),
            'member_of_specific_item' => __('Member of specific item', 'text-domain'),
            'member_of_any_items'     => __('Member of any item', 'text-domain'),
        ],
        'support_any_items'      => true,
        'has_any_items_callback' => [$this, 'has_any_items_callback'],
    ]);
}
```

**Callback Signatures**:
```php
// Get paginated items list
public function items_callback($search = '', $page = 1, $limit = 20) {
    return [
        $id => ['id' => $id, 'name' => $name, 'link' => $admin_url],
    ];
}

// Get single item (returns false if not available)
public function item_callback($item_value) {
    return ['id' => $id, 'name' => $name, 'link' => $url] | false;
}

// Get users for condition (paginated)
public function users_callback($data, $page = 1, $per_page = 10) {
    // $data contains: sub_condition, item_value, group_id, rounds_count
    return $this->return_users($user_ids);  // or return_wait($msg) or return_error($err)
}

// Check if user has any items
public function has_any_items_callback($user_id) {
    return true | false;
}
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/extending-access-controls/

---

### 2.8 TAB_BAR_SCREENS

**Purpose**: Add custom screens to the app's Tab Bar and More menu.

**Base Class**: `BuddyBossApp\Menus\Types\CoreAbstract`

**Implementation**:
```php
class MyScreens extends CoreAbstract {
    public function __construct() {
        parent::__construct();
    }

    public function setup() {
        // Optional: Register screen group
        $this->register_screen_group('my_group', __('My Group', 'text-domain'));

        // Register screens
        $this->register_screen(
            'screen_name',                    // name (lowercase, no spaces)
            __('Screen Label', 'text-domain'), // label
            ['type' => 'buddyboss', 'id' => 'icon-name'], // icon
            ['deeplink_slug' => 'my-screen'], // settings (optional)
            null,                              // menu_metabox_callback (optional)
            null,                              // menu_result_callback (optional)
            false                              // logged_in only (optional)
        );
    }
}

// Initialize
add_action('init', function() {
    MyScreens::instance();
});
```

**Note**: Requires corresponding screen creation in React Native app.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/adding-custom-screens-to-the-tab-bar/

---

### 2.9 DEEP_LINKING_PLUGIN

**Purpose**: Register URL patterns for native app navigation.

**Auto-Supported**: Custom post types and taxonomies are automatically supported.

**Response Format (CPT Single)**:
```json
{
    "action": "open_{postTypeSlug}",
    "namespace": "core",
    "url": "{url}",
    "item_id": "{itemID}",
    "_links": { ... }
}
```

**Available Filters**:
```php
// Change action value
add_filter('bbapp_deeplinking_cpt_action', function($action, $post) {
    return 'open_custom';
}, 10, 2);

// Change namespace
add_filter('bbapp_deeplinking_cpt_namespace', function($namespace, $post_type) {
    return 'my_namespace';
}, 10, 2);

// Add custom data
add_filter('bbapp_deeplinking_cpt', function($response, $post) {
    $response['custom_field'] = 'value';
    return $response;
}, 10, 2);

// Add embedded links
add_filter('bbapp_deeplinking_links', function($links, $data, $version) {
    $links['author'] = [['href' => $url, 'embeddable' => true]];
    return $links;
}, 10, 3);
```

**Custom Permalink Support**:
```php
// Extend TypeAbstract
class CustomType extends BuddyBossApp\DeepLinking\Type\TypeAbstract {
    public function parse($url) {
        // Parse URL and return deep link data
        return [
            'action'    => 'open_custom',
            'namespace' => 'my_namespace',
            'url'       => $url,
            'item_id'   => $id,
        ];
    }
}
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/extending-the-deep-linking-functionality/

---

### 2.10 GUTENBERG_BLOCKS_PLUGIN

**Purpose**: Create custom Gutenberg blocks for App Pages and App Editor.

**Base Class**: `BuddyBossApp\Admin\GutenbergBlockAbstract`

**Implementation**:
```php
class MyBlock extends GutenbergBlockAbstract {
    public function __construct() {
        $this->set_namespace('bbapp/my-block');
        $this->set_title(__('My Block', 'text-domain'));
        $this->set_description(__('Description', 'text-domain'));
        $this->set_icon('dashicons-admin-generic');
        $this->set_keywords(['keyword1', 'keyword2']);
        $this->set_attributes($this->get_attributes());
        $this->set_preview($this->get_preview());
        $this->init();
    }

    public function init() {
        parent::init();
        add_filter('bbapp_custom_block_data', [$this, 'update_block_data'], 10, 2);
    }

    public function get_attributes() {
        return [
            ['name' => 'title', 'fieldtype' => 'text', 'default' => 'Default', 'label' => 'Title'],
            ['name' => 'per_page', 'fieldtype' => 'number', 'default' => 5, 'label' => 'Items'],
        ];
    }

    public function get_preview() {
        return '<div>Preview HTML</div>';
    }

    public function get_results($attrs) {
        return []; // Direct results
    }

    public function update_block_data($app_page_data, $block_data) {
        // Add data source for app to fetch
        $app_page_data['data']['data_source'] = [
            'type'           => 'fetch',
            'request_params' => ['per_page' => 5],
            'route'          => '/wp/v2/my-endpoint',
        ];
        return $app_page_data;
    }
}
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/registering-custom-app-gutenberg-blocks/

---

### 2.11 WEB_FALLBACKS

**Purpose**: Edit theme templates for in-app browser display.

**Use Case**: Customize how web content appears when opened in the app's webview.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/editing-web-fallbacks-in-the-in-app-browser/

---

### 2.12 PROFILE_GROUP_TABS

**Purpose**: Control visibility and icons of profile/group tabs in the app.

**Available Filters**: Check documentation for specific filter names.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/profile-group-tabs-visibility-icons/

---

## 3. APP EXTENSIONS (React Native)

### 3.1 BEST_PRACTICES

**Directory Structure**:
```
custom_code/
├── index.js              # Entry point with applyCustomCode
├── components/           # UI components (JSX, styles)
└── containers/           # Redux, navigation logic
```

**Components Directory Rules**:
- Import from react-native (View, Text, etc.)
- Use React hooks (useState, useEffect, useRef)
- Import styles, types, reusable components

**Containers Directory Rules**:
- NO react-native UI imports
- Redux imports (useDispatch, useSelector)
- Navigation imports
- External library imports

**Import Aliases** (babel-plugin-module-resolver):
```javascript
// Instead of: import X from '../../../components/X'
import X from 'components/X'
```

**Code Standards**:
- Declare types (TypeScript/Flow)
- Separate styles from components
- Use functional components with hooks
- Use Redux + redux-saga for state/async
- Use axios for API calls
- Use reselect for selectors

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/app-development-best-practices/

---

### 3.2 SCREEN_MODIFICATION

**Purpose**: Modify existing screens without replacing entirely.

**Methods**:
- Use hooks system to inject content
- Override specific components
- Modify styles

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/making-changes-to-existing-app-screens/

---

### 3.3 SCREEN_CREATION

**Purpose**: Create new screens or replace existing ones.

**Entry Point**: `custom_code/index.js`

**Replace Existing Screen**:
```javascript
import MyCustomLoginScreen from './MyCustomLoginScreen';

export const applyCustomCode = externalCodeSetup => {
    externalCodeSetup.navigationApi.replaceScreenComponent(
        'LoginScreen',
        MyCustomLoginScreen
    );
};
```

**Add New Route**:
```javascript
import MyCustomScreen from './MyCustomScreen';

export const applyCustomCode = externalCodeSetup => {
    externalCodeSetup.navigationApi.addNavigationRoute(
        'customScreen',      // route name
        'MyCustomScreen',    // screen name
        MyCustomScreen,      // component
        'All'                // navigator: "Auth" | "noAuth" | "Main" | "All"
    );
};
```

**Post-Auth Landing Screen**:
```javascript
export const applyCustomCode = externalCodeSetup => {
    const { navigationApi } = externalCodeSetup;

    navigationApi.setFilterAfterAuthRoutes(afterAuthRoutes => ({
        ...afterAuthRoutes,
        'MyCustomScreen': MyCustomScreen
    }));

    navigationApi.setInitialAfterAuthRoute(props => 'MyCustomScreen');
};
```

**Screen Component Template**:
```javascript
import React from 'react';
import { View, Text } from 'react-native';

const MyCustomScreen = props => (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <Text>My Custom Screen</Text>
    </View>
);

export default MyCustomScreen;
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/creating-new-screens/

---

### 3.4 HOOKS_SYSTEM

**Purpose**: Insert content into specific areas of existing screens.

**Reference**: App Codex for available hook locations.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/using-hooks-in-the-app/

---

### 3.5 API_FETCHING

**Purpose**: Make network requests in custom screens.

**Methods**:
```javascript
// Using fetch
const response = await fetch(url);
const data = await response.json();

// Using axios
import axios from 'axios';
const { data } = await axios.get(url);

// Using BuddyBoss getApi
import { getApi } from '@src/services';
const api = getApi();
const response = await api.get(endpoint);
```

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/fetching-data-from-apis/

---

### 3.6 DATA_STORAGE

**Purpose**: Persist API responses using Redux store caching.

**Benefit**: Data reappears after app restart without re-fetching.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/data-storage-and-memory-management/

---

### 3.7 PUSH_NOTIFICATIONS_APP

**Purpose**: Display and handle custom push notifications in the app.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/displaying-custom-push-notifications/

---

### 3.8 DEEP_LINKING_APP

**Purpose**: Handle custom URL patterns and navigate to native screens.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/extending-the-deep-linking-functionality/

---

### 3.9 GUTENBERG_BLOCKS_APP

**Purpose**: Create React component representations of registered blocks.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/adding-custom-gutenberg-blocks-into-your-app/

---

### 3.10 NATIVE_LIBRARIES

**Purpose**: Install and use native modules in the app.

**Prerequisites**: Git repo connected, custom development setup complete.

**Documentation URL**: https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/installing-native-libraries/

---

## 4. CLASS_REFERENCE

| Class | Purpose | Section |
|-------|---------|---------|
| `BuddyBoss\Performance\Integration\Integration_Abstract` | API Caching | 2.2 |
| `BuddyBoss\Performance\Cache` | Cache operations | 2.2 |
| `BuddyBoss\Performance\Helper` | Cache settings | 2.2 |
| `BuddyBossApp\Notification\IntegrationAbstract` | Auto push notifications | 2.4 |
| `BuddyBossApp\Jobs` | Batch job processing | 2.4 |
| `BuddyBossApp\UserSegment\SegmentsAbstract` | Manual push segments | 2.5 |
| `BuddyBossApp\InAppPurchases\IntegrationAbstract` | In-app purchases | 2.6 |
| `BuddyBossApp\InAppPurchases\Orders` | Order management | 2.6 |
| `BuddyBossApp\AccessControls\Integration_Abstract` | Access controls | 2.7 |
| `BuddyBossApp\Menus\Types\CoreAbstract` | Tab bar screens | 2.8 |
| `BuddyBossApp\DeepLinking\Type\TypeAbstract` | Custom deep links | 2.9 |
| `BuddyBossApp\Admin\GutenbergBlockAbstract` | Gutenberg blocks | 2.10 |

---

## 5. HOOK_REFERENCE

| Hook | Type | Purpose |
|------|------|---------|
| `rest_api_init` | Action | Register REST routes, deep linking |
| `init` | Action | Initialize screen classes |
| `plugins_loaded` | Action | Load custom integrations |
| `save_post_{post_type}` | Action | Trigger on post save |
| `edit_post_{post_type}` | Action | Trigger on post edit |
| `trashed_post` | Action | Trigger on post trash |
| `bbapp_queue_task_{job_name}` | Action | Handle batch jobs |

---

## 6. FILTER_REFERENCE

| Filter | Purpose | Parameters |
|--------|---------|------------|
| `bbapp_admin_performance_custom_setting` | Add cache setting | `$settings` |
| `bbapp_custom_block_data` | Modify block data | `$app_page_data, $block_data` |
| `bbapp_deeplinking_cpt_action` | Change deep link action | `$action, $post` |
| `bbapp_deeplinking_cpt_namespace` | Change deep link namespace | `$namespace, $post_type` |
| `bbapp_deeplinking_cpt` | Add deep link data | `$response, $post` |
| `bbapp_deeplinking_links` | Add embedded links | `$links, $data, $version` |
| `bbapp_deeplinking_taxonomy_namespace` | Taxonomy namespace | `$namespace, $taxonomy` |
| `bbapp_deeplinking_taxonomy` | Taxonomy deep link | `$response, $taxonomy` |
| `bp_rest_{component}_prepare_value` | Modify BP API response | `$response, $object, $request` |
| `rest_prepare_{post_type}` | Modify WP API response | `$response, $post, $request` |

---

## 7. INTEGRATION_NAMES

**For Cache Purging and API References:**

| Name | Component |
|------|-----------|
| `app_page` | App Pages |
| `blog_post` | Blog Posts |
| `post_comment` | Post Comments |
| `categories` | Categories |
| `bp-activity` | BuddyBoss Activity |
| `bp-groups` | BuddyBoss Groups |
| `bp-members` | BuddyBoss Members |
| `bp-messages` | BuddyBoss Messages |
| `bp-notifications` | BuddyBoss Notifications |
| `bp-friends` | BuddyBoss Connections |
| `bbp-forums` | BuddyBoss Forums |
| `bbp-topics` | BuddyBoss Discussions |
| `bbp-replies` | BuddyBoss Replies |
| `bp-document` | BuddyBoss Documents |
| `bp-media-albums` | BuddyBoss Albums |
| `bp-media-photos` | BuddyBoss Photos |
| `sfwd-courses` | LearnDash Courses |
| `sfwd-courses-details` | LearnDash Course Details |
| `sfwd-lessons` | LearnDash Lessons |
| `sfwd-topic` | LearnDash Topics |
| `sfwd-quiz` | LearnDash Quizzes |

---

## 8. COMMON_PATTERNS

### Pattern: Plugin + App Feature Implementation

```
1. PLUGIN SIDE (PHP)
   ├── Create API endpoint (if needed)
   ├── Register with appropriate abstract class
   ├── Implement required methods
   └── Hook to WordPress actions/filters

2. APP SIDE (React Native)
   ├── Create component in custom_code/
   ├── Import in index.js
   ├── Register with externalCodeSetup
   └── Fetch data from API endpoint
```

### Pattern: Custom Screen with API Data

```php
// 1. Plugin: Create endpoint
add_action('rest_api_init', function() {
    register_rest_route('my-plugin/v1', '/data', [
        'methods'  => 'GET',
        'callback' => 'get_data',
    ]);
});
```

```javascript
// 2. App: Create screen
import React, { useEffect, useState } from 'react';
import { View, Text, FlatList } from 'react-native';

const MyScreen = () => {
    const [data, setData] = useState([]);

    useEffect(() => {
        fetch('https://site.com/wp-json/my-plugin/v1/data')
            .then(r => r.json())
            .then(setData);
    }, []);

    return (
        <FlatList
            data={data}
            renderItem={({ item }) => <Text>{item.title}</Text>}
        />
    );
};

// 3. App: Register screen
export const applyCustomCode = externalCodeSetup => {
    externalCodeSetup.navigationApi.addNavigationRoute(
        'myScreen', 'MyScreen', MyScreen, 'All'
    );
};
```

```php
// 4. Plugin: Add to tab bar
class MyScreenMenu extends CoreAbstract {
    public function setup() {
        $this->register_screen('myScreen', __('My Screen', 'td'), 'icon');
    }
}
```

### Pattern: Singleton Instance

```php
class MyClass {
    private static $instance = null;

    public static function instance() {
        if (null === self::$instance) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    private function __construct() {
        // Initialize
    }
}
```

---

## QUICK REFERENCE URLS

| Resource | URL |
|----------|-----|
| Plugin Extension Docs | https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app-plugin/ |
| App Extension Docs | https://buddyboss.com/resources/dev-docs/app-development/extending-the-buddyboss-app/ |
| App Codex | https://buddyboss.com/resources/app-codex/ |
| BuddyBoss App API | https://buddyboss.com/resources/api/app/ |
| BuddyBoss Platform API | https://buddyboss.com/resources/api/ |
| Code Reference | https://buddyboss.com/resources/reference/ |
| Hooks Reference | https://buddyboss.com/resources/reference/hooks/ |
| Functions Reference | https://buddyboss.com/resources/reference/functions/ |
| GitHub | https://github.com/buddyboss/buddyboss-platform |
