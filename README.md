# Store App - The successor of RouteRunner

This project is a re-write and expansion of my delivery app "RouteRunner"

## Things I wanna do differently

I want to write a api and then a frontend more seperately;
the previous code was hard to read and had some things I feel I could do better

I will probaly start with the manage part of the app and then move on to driver.

I plan to also make a store front page where customers can order products and a date and a slot for the delivery.

Ideally make is stateless so I could then make the app into different web apps (seperate worker for api, frontend etc..)

The orders can find the most efficient van between the 2, if there are 2 routes; 2 vans; and theres new order It should automatically choose the best van with the orders that are already really close to it; if the order can't fit in either van it gets rejected

## Ideas

### Less accuracy of the time of the order for cheaper price

Customer can choose a more costly option for accuracy; for example; strictly want it between `10:00 - 12:00`
Then the order gets added to `10:00 - 12:00` slot, If the customer wants to save money and doesn't care for when the delivery comes, they can choose a range; for example `10:00 - 16:00`, which would be much cheaper for them; then the system inserts it into a most efficient slot (least amount of additional driving distance) The order can come and go to the different slot to accomodate for the most efficient route, but 2 hours before the earliest slot it can go into (in this example it would be `8:00`), it locks into a single slot and stays there, can't move; the customer gets notified which slot the delivery is going to be out.

There are a few problems with this approach though; because the order has to be picked before the 2h of the earliest slot; they have to be only picked when the system has decided what slot it goes to; so they should really be done last; when the system already chose the slot

### Store only serves specific postcodes

Each store only covers certain postcodes; the customer's address gets matched by its postcode to whichever store owns it, and a postcode only ever belongs to one store, so there's no overlap and no picking between two stores for the same area; if the postcode isn't covered by any store then delivery just isn't available there yet.

### Address matching for guests

Because guests can type their address differently every time; for example `30 High Street, KY1 1AB` one order and `30 HighStreets, KY1 1AB` the next, matching on the text they typed won't tell us its the same property; so instead the address gets resolved through a lookup service which gives back a proper formatted address plus a stable id, that id is what decides if it's a new address or one we already have, not the raw text.

A few issues arise with this approach though; the lookup can fail for new builds or rural addresses that arent in the database yet, so there still needs to be a manual fallback for those, and that fallback won't dedupe as well since its just text again.

### Slot picking and holding a reservation

Before the customer can actually order, they put in their address which finds their store, then they see all the 2h slots for that day laid out in a row, for example `08:00-10:00`, `10:00-12:00`, `12:00-14:00` and so on, with the ones that don't have room crossed out. Picking a slot doesn't create the order straight away, it just holds that space for around 5 minutes while they finish checking out, and they can see they have a temporary hold with a countdown; if they don't finish in time the hold expires and the slot opens back up for someone else.

Problem that needs solving; two customers couldve seen the same slot as free at the same time, so the last bit of space in a slot has to be locked properly the moment someone reserves it or someone else could end up holding a slot that doesn't actually exist anymore.

### Picking doesn't need to wait for the slot to lock

We don't actually know the real crates for an order until someone picks it, for example the customer orders 2 milks but there's only 1 in stock so the picker has to swap it or mark it short. My first thought was elastic orders would have to be picked last since they could still move slots right up until locking, but thinking about it more, what gets picked doesn't actually depend on which slot or van the order ends up on, it's the same items either way; the only thing that actually depends on the slot is whether the real crate count fits the route its assigned to, and that's already handled by flagging it if it doesn't fit once its known.

So picking can just happen whenever theres staff free to do it, strict or elastic, doesn't matter; if most customers pick elastic slots that day it doesn't cause a crunch right before the vans leave, its just spread out through the day like normal, and the rare case where the picked crates don't fit the route it lands on gets caught and flagged for a reshuffle instead of trying to prevent it by holding picking back.

### Crate labels get printed at pick time, not lock time

Since picking now happens whenever, not just after locking, the crate label gets printed at pick time too; if the order is already locked into a slot (or was always strict, so it was never going anywhere), the label prints with the actual slot times on it already. If it isn't locked yet, the label just prints with a blank slot range, and an employee writes the slot on with a pen once the system locks it in later. Its so the driver can just look at the crate and know straight away which slot its going out for, without needing to look anything up.

### Estimated arrival only shown once the route is locked

The customer can see an estimated arrival time for their delivery, but only once their route has locked, since before that the stop couldve moved to a completely different van or slot and any eta shown earlier would just be wrong. Once it's locked the eta keeps updating live as the driver actually finishes each stop before it, so it gets more accurate through the day instead of being one static guess from the morning.

### Minimum order value

Orders under a certain amount, for example `£10`, shouldn't be allowed to go through, otherwise a van ends up driving out for just a couple items which isn't worth the trip. This gets checked before the customer is even let into checkout, and then checked again right when their reservation turns into a real order, in case they removed stuff from the cart in between.

## Structure


### Manage app


### Store Database

Entire database of the app it handles all the picking, driving, store front

- `stores` table
    - `id`: **Int** > primary_key
    - `address_id`: **Int** > foreign_key
    - `name`: **String**
    - `manager`: **String**
    - `minimum_order_value`: **Decimal** (checked before checkout, and again when a reservation converts to a real order)

- `serving_postcodes` table
    - `id`: **Int** > primary_key
    - `store_id`: **Int** > foreign_key
    - `postcode`: **String** > unique (the outward code only everything before the space, e.g. `GL1`, `SW1A`, `EH12`; each postcode maps to exactly one store, no overlap)

- `inventory` table
    - `store_id`: **Int** > primary_key, foreign_key
    - `product_id`: **Int** > primary_key, foreign_key
    - `quantity`: **Int**

- `addreses` table
    - `id`: **Int** > primary_key
    - `premise`: **String**
    - `street`: **String**
    - `postcode`: **String**
    - `door_lat`: **Decimal**
    - `door_lng`: **Decimal**
    - `park_lat`: **Decimal**
    - `park_lng`: **Decimal**
    - `external_place_id`: **String** > unique, nullable (UPRN or Google place_id)
    - `time_per_crate`:  **Int**      Seconds (rolling average, updated after each completed delivery at this address)


- `products` table
    - `id`: **Int** > primary_key
    - `name`: **String**
    - `barcode`: **String**
    - `is_age_restricted`: **Bool**
    - `unit_weight`: **Decimal**
    - `price`: **Decimal**
    - `type`: **String** Chilled, Grocery or frozen

- `customers` table
    - `id`: **Int** > primary_key
    - `first_name`: **String**
    - `middle_name`: **String** (nullable)
    - `last_name`: **String**
    - `phone_number`: **String**
    - `address_id`: **Int** > foreign_key
    - `email`: **String** > unique, nullable
    - `password_hash`: **String**, nullable (null for guest checkouts no account created)
    - `is_guest`: **Bool**
    - `email_verified_at`: **DATETIME**, nullable



- `orders` table
    - `id`: **Int** > primary_key
    - `order_ref`: **String** > unique (human-readable, e.g. `STOREXX-DATE-XXXX`)
    - `customer_id`: **Int** > foreign_key
    - `store_id`: **Int** > foreign_key
    - `grocery_crates`: **Int** (nullable, picker fills that in)
    - `chilled_crates`: **Int** (nullable, picker fills that in)
    - `frozen_crates`: **Int** (nullable, picker fills that in)
    - `total_crates`: **grocery_crates+chilled_crates+frozen_crates**
    - `slot_range_start`: **DATETIME**
    - `slot_range_end`: **DATETIME**
    - `is_locked`: **Bool** (locks 2h before earliest possible slot, can no longer move between routes)
    - `is_picked`: **Bool**
    - `picked_at`: **DATETIME** (nullable)
    - `picked_by`: **Int** > foreign_key to `users`, nullable
    - `delivered_by`: **Int** > foreign_key to `users`, nullable
    - `delivered_at`: **DATETIME** (nullable)
    - `is_delivered`: **Bool**
    - `proof_of_delivery`: **String** (nullable, file path/reference, S3)
    - `park_lat_actual`: **Decimal** (nullable, captured at delivery)
    - `park_lng_actual`: **Decimal** (nullable, captured at delivery)

- `order_items` table
    - `id`: **Int** > primary_key
    - `order_id`: **Int** > foreign_key
    - `product_id`: **Int** > foreign_key
    - `quantity`: **Int**
    - `price_at_order`: **Decimal**
    - `picked_quantity`: **Int**, nullable (actual quantity picked may be less than `quantity` if out of stock)
    - `is_substituted`: **Bool**
    - `substituted_product_id`: **Int** > foreign_key, nullable (alternative item the picker swapped in)

- `route_orders` table
    - `id`: **(route_id, position)** primary_key
    - `route_id`: **String** > foreign_key
    - `order_id`: **Int** > foreign_key
    - `position`: **Decimal**
    - `is_flagged`: **Bool** (checkout already checks fit before accepting an order, so this is for fit problems discovered *after* acceptance e.g. picked crates exceed the original estimate, or a locked route hits a real-world disruption needs manual review since the order can't just be un-accepted) - that shouldnt happen on customers order; if it cant fit it it shouldnt even accept it or give an option
    - `estimated_arrival`: **DATETIME** (cached, recomputed when the route changes or a prior stop completes; only exposed to the customer once the parent `delivery_routes.is_locked` is true)

- `delivery_routes` table
    - `id`: **String** > primary_key -> `&create_id`
    - `store_id`: **Int** > foreign_key
    - `slot_range_start`: **DATETIME**
    - `slot_range_end`: **DATETIME**
    - `van_id`: **Int** > foreign_key
    - `total_crates`: **Int**
    - `estimated_duration_seconds`: **Int** (cached, recomputed when stops change)
    - `planned_departure_at`: **DATETIME** (anchor for ETA chain before the van leaves)
    - `actual_departure_at`: **DATETIME** (nullable, set by driver app when van leaves store replaces planned_departure_at as the ETA anchor once known)
    - `is_locked`: **Bool** (finalised and visible to drivers, can't be re-optimised; also gates customer-facing `estimated_arrival` visibility)
    - `*create_id` -> `{store_id}-{slot_range_start.date}-{slot_range_start.time}-{slot_range_end.time}-{van_id}`

- `slot_reservations` table
    - `id`: **Int** > primary_key
    - `session_token`: **String** (anon customer/session id if no order exists yet)
    - `store_id`: **Int** > foreign_key
    - `address_id`: **Int** > foreign_key
    - `slot_range_start`: **DATETIME**
    - `slot_range_end`: **DATETIME**
    - `grocery_crates`: **Int**
    - `chilled_crates`: **Int**
    - `frozen_crates`: **Int**
    - `expires_at`: **DATETIME** (now + 5 min at creation)
    - `order_id`: **Int** > foreign_key, nullable (set once the reservation converts into a real order)

- `vans` table
    - `id`: **Int** > primary_key
    - `store_id`: **Int** > foreign_key 
    - `license_plate`: **String**
    - `capacity_grocery_crates`: **Int** default 3x3x5 - 45
    - `capacity_chilled_crates`: **Int**  default 3x2x5 - 30
    - `capacity_frozen_crates`: **Int** default 3x2x5 - 30
    - `status`: **String**                (on road off road)

- `drive_times` table
    - `id`: **Int** > primary_key
    - `origin_address_id`: **Int** > foreign_key
    - `dest_address_id`: **Int** > foreign_key
    - `duration_seconds`: **Int**
    - `distance_metres`: **Int**
    - `fetched_at`: **DATETIME**
    - unique on `(origin_address_id, dest_address_id)` — cache to avoid re-hitting the routing API for pairs already computed; when an address's coordinates get updated, delete its cached rows so they recompute instead of going stale

- `users` table
    - `id`: **Int** > primary_key
    - `username`: **String** > unique
    - `password_hash`: **String**
    - `is_active`: **Bool**
    - `role`: **String** (`manager` / `picker` / `driver`)