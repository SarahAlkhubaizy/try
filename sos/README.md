classDiagram
    class BaseModel {
        +String id (UUID4)
        +DateTime created_at
        +DateTime updated_at
        +save() void
        +update() void
        +delete() void
    }

    class User {
        +String first_name
        +String last_name
        +String email
        +String password
        +Boolean is_admin
        +register() Boolean
        +update_profile(first_name, last_name, email) Boolean
    }

    class Place {
        +String title
        +String description
        +Float price_per_night
        +Float latitude
        +Float longitude
        +String owner_id
        +List~String~ amenity_ids
        +create_place() Place
        +update_details(title, description, price, latitude, longitude) Boolean
        +add_amenity(amenity_id) void
        +get_reviews() List~Review~
    }

    class Review {
        +String place_id
        +String user_id
        +Int rating
        +String comment
        +create_review() Review
        +update_review(rating, comment) Boolean
    }

    class Amenity {
        +String name
        +String description
        +create_amenity() Amenity
        +update_amenity(name, description) Boolean
    }

    %% علاقات الميراث (Inheritance)
    User --|> BaseModel
    Place --|> BaseModel
    Review --|> BaseModel
    Amenity --|> BaseModel

    %% العلاقات والروابط بين الكيانات (Associations & Aggregations)
    User "1" --> "0..*" Place : owns (صاحب العقار)
    User "1" --> "0..*" Review : writes (كاتب التقييم)
    Place "1" *-- "0..*" Review : contains (يحتوي التقييمات)
    Place "0..*" o-- "0..*" Amenity : has (يمتلك مرافق)