
### Customer Store Front

### `/`

Displays the main page of the store, shows tranding products,  in nav bar you can enter postcode and it will set your location to that postcode, it then redicrects to `/?store=<storeid>/`, if you want to add an item to a cart it first asks for a postcode

### `/?store=<storeid>`

Displays the page similar to that main page but with that stores specific stock, you can add products to cart and then check out where you put your full address, if the postcode provided is owned by another store, highlight the products not in stock in that store

### `/myaccount/`

Displays the information of the customer, previous orders, preferences, change details etc...

### `/checkout/`

Checkout page with all the details about the order, login is required and customer chooses adress from their address list, prompt to create if none exists pressing checkout creates the order at specific timeslot or range provided

### `/login/`

A page to log in

### `/register/`

A page to register

### The data it needs, writes to database (Database API)

1. `GET` List of available products (all the stores)
2. `GET` Images of those products
3. `GET` Available slots
4. `POST` Reserve a slot
5. `POST` Create order from the reservation
6. `GET` Store id by customer postcode (looks up which store owns that postcode)
7. `GET` Store inventory by id
8. `POST` Create customer account
9. `POST` Change customer info, alternetive preference, add address etc..
10. `GET` Customer account information, order history, details
11. `POST` Customer login
12. `GET` Order status

### The geolocation data

1. `POST` Get Address info by the fields the customer provided 

### 