### Dynamic E-Commerce Collection Page in FlutterFlow (Shopify GraphQL)

A complete architectural pattern for building a responsive e-commerce Product Collection page in FlutterFlow connected to the Shopify Storefront GraphQL API. It fetches both standard paginated products and curated featured products from a collection metafield, while handling loading states, empty collections, and low-inventory catalogs without layout shifts or null-check crashes.

#### Key Benefits

* **Crash-Free Empty State Handling:** Prevents null-pointer exceptions when Shopify returns empty arrays (`[]`) for new or unpopulated collections.
* **Zero Dead-End Shopping:** Replaces blank screens with a branded empty-state card and category navigation CTA when a collection has 0 products.
* **Polished Low-Inventory UX:** Displays a lightweight "More Coming Soon" footer banner for smaller collections (1–19 items) so short grids feel intentional rather than cut off.
* **Dynamic Merchandising:** Automatically hides the "Featured Highlights" carousel when no featured metafield products are assigned to a collection.
* **Smooth Cursor Pagination:** Supports infinite/load-more pagination using Shopify's `endCursor` and `hasNextPage` fields without redundant API calls.

---

### Step 1: Configure the Shopify Storefront GraphQL Query

> **Why we do this:** Instead of making two separate network requests (one for highlighted products and one for the regular product grid), a single GraphQL query fetches both at the same time—making the page load faster and keeping your API logic simple.

Create an API call (e.g., `GetCollectionProductsCall`) to fetch the collection's featured metafield references alongside its first page of products:

```graphql
query GetCollectionProducts($collectionId: ID!, $first: Int = 20, $cursor: String) {
  node(id: $collectionId) {
    ... on Collection {
      id
      title
      featuredProductsMetafield: metafield(namespace: "custom", key: "featured_products") {
        references(first: 10) {
          nodes {
            ... on Product {
              id
              title
              images(first: 1) {
                edges {
                  node {
                    url
                  }
                }
              }
              variants(first: 1) {
                edges {
                  node {
                    price {
                      amount
                    }
                  }
                }
              }
            }
          }
        }
      }
      products(first: $first, after: $cursor) {
        edges {
          node {
            id
            title
            images(first: 1) {
              edges {
                node {
                  url
                }
              }
            }
            variants(first: 1) {
              edges {
                node {
                  price {
                    amount
                  }
                }
              }
            }
          }
        }
        pageInfo {
          endCursor
          hasNextPage
        }
      }
    }
  }
}

```

---

### Step 2: Set Up Page Parameters & Page State Variables

> **Why we do this:** Page Parameters tell the screen *which* category the shopper tapped on, while Page State variables act like short-term memory—tracking whether the page is still loading, storing the products fetched so far, and remembering where to resume when loading the next batch.

On your `ProductCollection` page, define the following parameters and local state variables:

1. **Page Parameters:**
* `collectionId` (`String`, Required) — The Shopify Global ID (e.g., `gid://shopify/Collection/123456789`).
* `collectionName` (`String`, Required) — Display title for the app bar and empty-state copy.


2. **Page State Variables:**
* `productList` (`List<JSON>`, Initial Value: `[]`) — Holds the merged list of loaded product nodes.
* `endCursor` (`String`, Nullable) — Stores the pagination cursor for loading additional pages.
* `hasNextPage` (`Boolean`, Initial Value: `false`) — Controls visibility of the "Show More" pagination button.
* `isLoading` (`Boolean`, Initial Value: `true`) — Prevents premature UI rendering before the initial API call resolves.



---

### Step 3: Build the Reusable Empty-State Component

> **Why we do this:** If a customer opens a brand-new category that doesn't have products yet, showing a blank white screen makes the app look broken. Building a dedicated card gives shoppers a friendly message and a clear button to keep browsing elsewhere.

Create a standalone component (e.g., `EmptyCollectionCard`) to display when a collection has zero products:

1. Add a **Component Parameter**: `collectionName` (`String`).
2. Design a centered, rounded container (`16px` border radius, `24px` padding) containing:
* A circular icon badge (`64x64`) with a soft accent tint and an inventory/shopping icon.
* **Headline:** *"Curating This Collection"*
* **Dynamic Subtext (Combine Text):** *"We’re currently adding new products to `[collectionName]`. Check back soon to explore the latest arrivals!"*
* **Primary CTA Button:** *"Browse All Categories"* wired to navigate back to your main Categories screen.



---

### Step 4: Structure the Widget Tree & Conditional Visibility

> **Why we do this:** Conditional visibility acts like smart switches that show or hide sections based on what data actually arrived—hiding empty carousels, preventing layout glitches while data loads, and reassuring shoppers when a smaller category only has a few items.

Keep a fixed top header and bottom navigation bar, and wrap the main content area inside an `Expanded` → `SingleChildScrollView` → `Column`:

1. **Top App Bar (Fixed):** Back button, dynamic `collectionName` title, and search icon button.
2. **Main Scrollable Body (`SingleChildScrollView`):**
* **Conditional Visibility:** `isLoading == false`
* **Child 1 — Featured Highlights Section (`Column`):**
* **Conditional Visibility:** JSON Path `$.data.node.featuredProductsMetafield.references.nodes[0]` **Is Set and Not Empty** (`!= null`). *(Checking index `[0]` ensures the section hides cleanly when Shopify returns an empty array `[]`).*
* Contains the section title and a horizontal `ListView` (`height: 300`, `shrinkWrap: true`) rendering product cards.


* **Child 2 — "All Products" Header (`Row`):**
* **Conditional Visibility:** `productList` **Is Not Empty**.


* **Child 3 — 2-Column Product Grid (`GridView`):**
* **Conditional Visibility:** `productList` **Is Not Empty**.
* Configure `crossAxisCount: 2`, `shrinkWrap: true`, and `NeverScrollableScrollPhysics` so it scrolls smoothly inside the parent `SingleChildScrollView`.


* **Child 4 — Pagination Button (*"Show More Products ↓"*):**
* **Conditional Visibility:** `isLoading == false` **AND** `productList` **Is Not Empty** **AND** `hasNextPage == true`.


* **Child 5 — Low-Inventory Footer Banner:**
* **Conditional Visibility:** `isLoading == false` **AND** `productList -> Number of Items > 0` **AND** `productList -> Number of Items < 20`.
* A compact horizontal bordered card with a subtle icon badge and text: *"More items are being added to this collection soon. Stay tuned!"*


* **Child 6 — Empty Collection Card (`EmptyCollectionCard`):**
* **Conditional Visibility:** `isLoading == false` **AND** `productList` **Is Empty**.
* Pass `widget.collectionName` into the component parameter.





---

### Step 5: Configure the `On Page Load` Action Flow

> **Why we do this:** When Shopify has 0 products in a category, it sends back an empty list (`[]`) instead of "nothing" (`null`). Checking if the very first item (`edges[0]`) exists before reading the list prevents the app from crashing mid-load and guarantees `isLoading` always turns off properly.

In the page's **On Page Load** action editor, configure the following sequence:

1. **Update Page State:** Reset `productList = []`, `endCursor = null`, `hasNextPage = false`, and `isLoading = true`.
2. **Backend Call:** Execute `GetCollectionProductsCall` passing `collectionId`.
3. **Conditional Action (`if / else`):**
* **Condition:** Check if `firstPageResponse` JSON Path `$.data.node.products.edges[0]` **Is Set and Not Empty** (`!= null`). *(Checking `[0]` prevents null-check crashes on empty arrays).*
* **TRUE Branch (Products Exist):**
* Update `productList` with JSON Path `$.data.node.products.edges[*].node` (List).
* Update `endCursor` with JSON Path `$.data.node.products.pageInfo.endCursor`.
* Update `hasNextPage` with JSON Path `$.data.node.products.pageInfo.hasNextPage`.


* **FALSE Branch (0 Products):**
* Update `productList = []` and `hasNextPage = false`.




4. **Update Page State (After Conditional Merge):** Set `isLoading = false` so the appropriate UI state renders immediately.

---

### Step 6: Wire Up "Show More Products" Pagination

> **Why we do this:** Loading 20 products at a time instead of hundreds at once keeps the app fast and saves mobile data. Using the saved `endCursor` bookmark tells Shopify to hand over only the next 20 items and seamlessly attach them to the bottom of the existing grid.

On the **Show More Products ↓** button's `On Tap` action:

1. **Backend Call:** Call `GetCollectionProductsCall` passing `collectionId` and `cursor: endCursor`.
2. **Update Page State:**
* Append the new `$.data.node.products.edges[*].node` items to `productList` (using a list-merge custom function or FlutterFlow's `Add to List`).
* Update `endCursor` and `hasNextPage` from the new response's `pageInfo`.


3. **Scroll To:** Optionally animate the `SingleChildScrollView` controller slightly downward to reveal the newly loaded row of products.

> NOTE
> **Haven't set up the `custom.featured_products` metafield on your Shopify collections yet?**  
> Follow this step-by-step guide to configure collection-level featured products in Shopify (no paid apps required): [Shopify Dynamic Featured Products Guide](https://github.com/sandeepsingh-tech/shopify-dynamic-featured-products)
