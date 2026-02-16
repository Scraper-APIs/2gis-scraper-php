# 2GIS Scraper PHP

**English** | [Русский](https://github.com/Scraper-APIs/2gis-parser-php)

A PHP client library for scraping data from [2GIS](https://2gis.ru) — the largest business directory and map service across Russia, Central Asia, and the Middle East. Extract places, reviews, real estate listings, and job vacancies with typed DTOs.

Powered by [Apify](https://apify.com/) actors under the hood.

## Installation

```bash
composer require scraper-apis/2gis-parser
```

## Quick Start

```php
use TwoGisParser\Client;

$client = new Client('your_apify_api_token');

// Search for restaurants in Moscow
$places = $client->scrapePlaces(
    query: ['restaurant'],
    location: 'Moscow',
    maxResults: 50,
);

foreach ($places as $place) {
    echo "{$place->name} — {$place->address}\n";
    echo "Rating: {$place->rating} ({$place->reviewCount} reviews)\n";

    if ($place->hasContactInfo()) {
        echo "Phone: {$place->getFirstPhone()}\n";
        echo "Email: {$place->getFirstEmail()}\n";
    }

    if ($place->hasWebsite()) {
        echo "Website: {$place->website}\n";
    }
}
```

## Available Methods

The client wraps 4 specialized Apify actors, each optimized for a specific data type.

### 1. Places / Businesses

Search the 2GIS catalog by keyword and city. Returns rich records with 60+ fields including contacts, ratings, schedule, photos, and more. Covers 207 cities across 20 countries.

```php
use TwoGisParser\Language;
use TwoGisParser\Country;

$places = $client->scrapePlaces(
    query: ['dentist', 'dental clinic'],
    location: 'Saint Petersburg',
    maxResults: 200,
    language: Language::Russian,
    country: Country::Russia,
    options: [
        'categoryFilter' => ['Dentistry'],
        'filterRating' => 'excellent',       // 4.5+ rating
        'skipClosedPlaces' => true,
        'filterCardPayment' => true,
        'maxReviews' => 10,                  // fetch up to 10 reviews per place
        'maxPhotos' => 5,                    // fetch up to 5 photos per place
    ],
);
```

**Available options for `scrapePlaces()`:**

| Option | Type | Description |
|--------|------|-------------|
| `categoryFilter` | `string[]` | Filter by business categories (fuzzy-matched against 731 categories) |
| `filterRating` | `string` | Min rating: `perfect` (4.9+), `excellent` (4.5+), `pretty_good` (4.0+), `nice` (3.5+), `not_bad` (3.0+) |
| `sortBy` | `string` | Sort by: `rating`, `opened_time`, `name` |
| `filterWebsite` | `string` | `all`, `withWebsite`, `withoutWebsite` |
| `skipClosedPlaces` | `bool` | Skip permanently closed places |
| `searchMatching` | `string` | `all`, `only_includes`, `only_exact` |
| `filterHasPhotos` | `bool` | Only places with photos |
| `filter24h` | `bool` | Only 24/7 places |
| `filterNewPlaces` | `bool` | Only recently opened places |
| `filterDelivery` | `bool` | Has delivery |
| `filterTakeaway` | `bool` | Has takeaway |
| `filterCardPayment` | `bool` | Accepts card payment |
| `filterWifi` | `bool` | WiFi available |
| `filterHasGoods` | `bool` | Has price list/goods on 2GIS |
| `filterAvgPriceMin` | `int` | Min average check (local currency) |
| `filterAvgPriceMax` | `int` | Max average check (local currency) |
| `maxReviews` | `int` | Max reviews per place (0 = disabled, 99999 = all) |
| `reviewsRating` | `string` | `all`, `positive` (4-5), `negative` (1-2) |
| `reviewsWithAnswer` | `bool` | Only reviews with business reply |
| `reviewsStartDate` | `string` | Only reviews after YYYY-MM-DD |
| `reviewsTopic` | `string` | Server-side topic filter |
| `reviewsFilterKeyword` | `string` | Client-side text search within reviews |
| `reviewsSource` | `string` | `all`, `2gis`, `flamp`, `booking` |
| `maxPhotos` | `int` | Max photos per place (0 = disabled, 99999 = all) |
| `photoCategories` | `string[]` | Photo categories: `food_and_drinks`, `interior`, `outside`, `price_list_image`, etc. |
| `scrapeOrgBranches` | `bool` | Scrape all branches of found organizations |

**Place DTO helpers:**

```php
$place->hasContactInfo();    // true if phones or emails exist
$place->getFirstPhone();     // first phone number or null
$place->getFirstEmail();     // first email or null
$place->hasWebsite();        // true if website is set
$place->getCoordinates();    // ['lat' => float, 'lng' => float] or null
```

### 2. Reviews

Extract reviews from specific 2GIS places. Accepts place URLs, search URLs, or branch IDs. Returns flat one-row-per-review records, ideal for CSV/Excel export. Aggregates reviews from 2GIS, Flamp, and Booking.

```php
use TwoGisParser\ReviewsRating;
use TwoGisParser\ReviewsSource;

$reviews = $client->scrapeReviews(
    startUrls: [
        'https://2gis.ru/moscow/firm/70000001057394703',
        '70000001057394704',  // plain branch ID also works
    ],
    maxReviews: 500,
    reviewsRating: ReviewsRating::Negative,
    reviewsSource: ReviewsSource::TwoGis,
    options: [
        'reviewsStartDate' => '2025-01-01',
        'reviewsWithAnswer' => true,
    ],
);

foreach ($reviews as $review) {
    echo "{$review->authorName}: {$review->rating}/5\n";
    echo "{$review->text}\n";

    if ($review->hasOfficialAnswer()) {
        echo "Reply: {$review->getOfficialAnswerText()}\n";
    }
}
```

**Available options for `scrapeReviews()`:**

| Option | Type | Description |
|--------|------|-------------|
| `reviewsWithAnswer` | `bool` | Only reviews with business reply |
| `reviewsStartDate` | `string` | Only reviews after YYYY-MM-DD |
| `reviewsTopic` | `string` | Server-side topic filter |
| `reviewsFilterKeyword` | `string` | Client-side keyword filter |

**Review DTO helpers:**

```php
$review->hasOfficialAnswer();      // true if business replied
$review->getOfficialAnswerText();  // reply text or null
$review->hasPhotos();              // true if review has photos
$review->isPositive();             // true if rating >= 4
$review->isNegative();             // true if rating <= 2
```

### 3. Real Estate / Property

Scrape property listings from 2GIS across Russia, Kazakhstan, and Kyrgyzstan. Supports sale/rent for residential and commercial properties, plus daily rentals.

```php
use TwoGisParser\PropertyCategory;
use TwoGisParser\PropertySort;

$properties = $client->scrapeProperties(
    location: 'Kazan',
    maxResults: 500,
    category: PropertyCategory::SaleResidential,
    sort: PropertySort::PriceAsc,
    options: [
        'rooms' => ['2', '3'],
        'priceMax' => 15000000,
        'areaMin' => 50,
        'notFirstFloor' => true,
        'notLastFloor' => true,
    ],
);

foreach ($properties as $property) {
    echo "{$property->name}\n";
    echo "Price: {$property->getPriceFormatted()}\n";
    echo "Area: {$property->area} m², floor {$property->floor}\n";
    echo "Address: {$property->address}\n";
}
```

**Available options for `scrapeProperties()`:**

| Option | Type | Description |
|--------|------|-------------|
| `rooms` | `string[]` | Room count: `studio`, `1`, `2`, `3`, `4`, `5` |
| `propertyType` | `string[]` | `flat`, `house`, `land`, `cottage`, `room`, `townhouse`, `share`, `part_house` |
| `newBuilding` | `string[]` | `secondary` (resale), `new` (new building) |
| `priceMin` | `int` | Minimum price |
| `priceMax` | `int` | Maximum price |
| `pricePerMeterMin` | `float` | Min price per m2 |
| `pricePerMeterMax` | `float` | Max price per m2 |
| `areaMin` | `float` | Min area in m2 |
| `areaMax` | `float` | Max area in m2 |
| `floorMin` | `int` | Min floor number |
| `floorMax` | `int` | Max floor number |
| `notFirstFloor` | `bool` | Exclude first floor |
| `notLastFloor` | `bool` | Exclude last floor |
| `floorsInBuildingMin` | `int` | Min building floors |
| `floorsInBuildingMax` | `int` | Max building floors |
| `metroTime` | `string` | Metro proximity: `5`, `10`, `20` (minutes walking) |
| `provider` | `string[]` | Listing providers: `cian`, `domclick` |

**Property DTO helpers:**

```php
$property->hasImages();         // true if images array is not empty
$property->getCoordinates();    // ['lat' => float, 'lng' => float] or null
$property->getPriceFormatted(); // "1,500,000 RUB" or null
```

### 4. Jobs / Vacancies

Scrape job vacancies from 2GIS by city. Supports filtering by 224 job categories and salary range.

```php
$jobs = $client->scrapeJobs(
    location: 'Novosibirsk',
    maxResults: 300,
    salaryMin: 80000,
    categoryId: '200',  // Developer
);

foreach ($jobs as $job) {
    echo "{$job->name} at {$job->orgName}\n";
    echo "Salary: {$job->salaryLabel}\n";
    echo "Address: {$job->address}\n";

    if ($job->hasApplyUrl()) {
        echo "Apply: {$job->applyUrl}\n";
    }
}
```

**Job DTO helpers:**

```php
$job->hasApplyUrl();      // true if apply URL is set
$job->getCoordinates();   // ['lat' => float, 'lng' => float] or null
```

## Enums Reference

### Language

Supported across all 4 actors. 14 languages covering the 2GIS service area.

| Case | Value | Language |
|------|-------|----------|
| `Auto` | `auto` | Auto-detect |
| `Russian` | `ru` | Russian |
| `English` | `en` | English |
| `Arabic` | `ar` | Arabic |
| `Kazakh` | `kk` | Kazakh |
| `Uzbek` | `uz` | Uzbek |
| `Kyrgyz` | `ky` | Kyrgyz |
| `Armenian` | `hy` | Armenian |
| `Georgian` | `ka` | Georgian |
| `Azerbaijani` | `az` | Azerbaijani |
| `Tajik` | `tg` | Tajik |
| `Czech` | `cs` | Czech |
| `Spanish` | `es` | Spanish |
| `Italian` | `it` | Italian |

### Country

21 countries where 2GIS operates.

| Case | Value |
|------|-------|
| `Auto` | *(auto-detect)* |
| `Russia` | `ru` |
| `Kazakhstan` | `kz` |
| `UAE` | `ae` |
| `Uzbekistan` | `uz` |
| `Kyrgyzstan` | `kg` |
| `Armenia` | `am` |
| `Georgia` | `ge` |
| `Azerbaijan` | `az` |
| `Belarus` | `by` |
| `Tajikistan` | `tj` |
| `SaudiArabia` | `sa` |
| `Bahrain` | `bh` |
| `Kuwait` | `kw` |
| `Qatar` | `qa` |
| `Oman` | `om` |
| `Iraq` | `iq` |
| `Chile` | `cl` |
| `Czechia` | `cz` |
| `Italy` | `it` |
| `Cyprus` | `cy` |

### Other Enums

| Enum | Values |
|------|--------|
| `RatingFilter` | `None`, `Perfect` (4.9+), `Excellent` (4.5+), `PrettyGood` (4.0+), `Nice` (3.5+), `NotBad` (3.0+) |
| `ReviewsRating` | `All`, `Positive` (4-5 stars), `Negative` (1-2 stars) |
| `ReviewsSource` | `All`, `TwoGis`, `Flamp`, `Booking` |
| `PropertyCategory` | `SaleResidential`, `SaleCommercial`, `RentResidential`, `RentCommercial`, `DailyRent` |
| `PropertySort` | `Default`, `PriceAsc`, `PriceDesc`, `AreaAsc`, `AreaDesc` |

## Configuration

```php
use TwoGisParser\Client;
use TwoGisParser\Config;

// Simple — just pass the API token
$client = new Client('your_apify_api_token');

// Advanced — override timeout or base URL
$client = new Client('token', new Config(
    apiToken: 'token',
    baseUrl: 'https://api.apify.com/v2',
    timeout: 600,
));
```

## Error Handling

```php
use TwoGisParser\Exception\ApiException;
use TwoGisParser\Exception\RateLimitException;

try {
    $places = $client->scrapePlaces(query: ['cafe'], location: 'Almaty');
} catch (RateLimitException $e) {
    // Retry after the suggested delay
    sleep($e->retryAfter);
} catch (ApiException $e) {
    echo "API error: {$e->getMessage()}\n";
}
```

## Requirements

- PHP 8.3+
- [Apify API token](https://console.apify.com/account/integrations)

## License

MIT
