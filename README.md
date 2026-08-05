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

### Checking a slot and reserving a slot arent the same thing

When the customer is just looking at which slots are open, that doesn't need to write anything anywhere; its just adding up whats already been reserved or committed on each van for that slot and seeing if theres room, so it can just be a read. The actual reserving only happens once they pick one, thats the bit that has to insert the row and lock things properly so two people cant grab the same last space.

I got a bit confused on this myself at first, kept thinking the availability check had to insert something to "know" the slots, but it doesn't, its just reading the same data a reservation would read, reservation is just the one that adds to it.

### The fit check has two parts, crates and time

Checking if an order fits isn't just about crates, a van could have room left in its capacity but still not have enough time left in the slot to actually get there and back on schedule; so the check also has to try inserting the new stop into each vans current route using the cached drive times, and see if it still gets everyone there inside their slot and the whole route still fits the 2h window.

### Optimisation queue only makes it better, never breaks a promise

Whether an order fits has to be answered straight away when the customer is checking out, so that part cant go through a queue, it has to be a proper fit check (crates and time) done there and then; but working out the *best* order to visit everyone in, across all the vans, is the heavy bit and that can happen after, on a queue with a worker. Since the customer already got told yes based on a placement that actually works, the worker reshuffling things afterwards can only make the route more efficient, it can never turn a slot that fit into one that doesn't, so it doesn't matter if that queue is a bit slow or backed up.

### Looking up an address properly instead of typing it

Instead of letting the customer type their whole address which is what caused the duplicate address problem in the first place, they just type their postcode and get shown the list of premises on that postcode from a lookup service, then pick theirs from the list; picking one resolves to a full address with the coordinates and the stable id from the lookup service, which is what actually gets used later for the drive time checks and for figuring out if its an address we already have.

