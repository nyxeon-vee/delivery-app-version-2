# Geolocation API

This api is going to use another geolocating API to get coordinates for the addresses it first checks if addresses exist in the database

`GET /geo/?postcode=A124AA `->
```
{
    "premises":[
        "2","4","6","8","10","12","14"
    ]
    
}
```

`GET /geo/?postcode=A124AA&door=10` ->
```
{
    "address":{
        "premise": "10"
        "street": "High Street"
        "postcode": "A12 4AA"
        "external_place_id": "uusu-dsd09-3d93d-32343-Examp"
        "door_lat": "1.0300"
        "door_lng"" "4.0100"
        etc..
    }
}
```