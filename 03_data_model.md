# Data Model

Here’s a breakdown of the data that powers the Sharing Excess tech platform. Each header represents an individual table in our PostgreSQL database, below which you’ll find a literal data type definition, a list of all of the associated properties, and an explanation of what each individual property means.

## `rescue`

### Definition
A rescue represents the complete journey of food rescued by Sharing Excess, from donor(s) to recipient(s). Each rescue is the “parent” to a list of individual “transfers”, which represent individual collections or distributions of food from an individual organization. The total weight of all the “collection” transfers in a rescue should equal the total weight of all “distribution” transfers, meaning we always distribute all food we collect.

### Properties
```
id: string (primary key)
handler_email: string (foreign key: users.email) (nullable)
wholesale_market_id: string (foreign key: wholesale_markets.id) (nullable)
type: enum ("retail", "wholesale", "direct_link")
status: enum ("scheduled", "active", "completed", "cancelled")
notes: string (nullable)
timestamp_created: date
timestamp_updated: date
timestamp_scheduled: date
timestamp_completed: date (nullable)
```

### Details
- **id**: The unique identifier for the rescue, which serves as the primary key.
- **handler_email**: The email of the handler responsible for the rescue, which references the `users.email` field. Rescues can be created without a handler_email, meaning that they are “available” to be claimed by a driver (though this is infrequently used, and would only apply to “retail” type rescues).
- **wholesale_market_id**: The ID of the wholesale market associated with the rescue, which is nullable and references the `wholesale_markets.id` field. This should always be populated for “wholesale” type rescues, but could also theoretically be populated for a retail or direct_link rescue that moves through a wholesale market. We use this in our “impact” calculations to demonstrate the impact of a given market (and this field allows us to indicate that a market was included in a rescue, even if the food didn’t come directly from a market vendor).
- **type**: The type of the rescue, which can be one of the following values: `"retail"`, `"wholesale"`, `"direct_link"`.
    - “**Retail**” means that the rescue was performed by a Sharing Excess driver, who travelled to each organization in the transfer list, and collected/distributed the food directly.
    - “**Wholesale**” means that the rescue took place in a wholesale market, where the collected food was delivered *to us*, and we coordinate any necessary sorting and distribution (we may drive the food to a recipient, but they may also pick it up directly).
    - “**Direct link**” means that we coordinated, but did not handle the transfer of food. This most commonly looks like a farm or distributor reaching out to us to find a recipient for a large load, at which point we connect them with a local recipient (and may or may not arrange for freight transit).
- **status**: The status of the rescue, which can be one of the following values: `"scheduled"`, `"active"`, `"completed"`, `"cancelled"`.
    - Note: “cancelled” rescues do not count towards impact metrics.
- **notes**: Additional notes about the rescue, which are nullable.
- **timestamp_created**: The timestamp when the rescue was created.
- **timestamp_updated**: The timestamp when the rescue was last updated.
- **timestamp_scheduled**: The timestamp when the rescue is scheduled (think the “start” time).
- **timestamp_completed**: The timestamp when the rescue was completed (think the “end” time), which is nullable (note: must be populated for rescues with status “completed”).

## `transfer`

### Defintion

A transfer represents the exchange of food items between Sharing Excess, and a partner organization. It defines what food was exchanged, when, and where. A transfer can be of type “collection” (food coming *in* from a donor) or “distribution” (food going *out* to a recipient). Transfers are always associated with a “parent” rescue, which allows us to link collections and distributions together.

As a note, we use transfers to calculate our cumulative impact by summing the “total_weight”. These sum totals can be filtered by type, organization, location, date, etc. 

### Properties

```
id: string (primary key)
rescue_id: string (foreign key: rescue.id)
handler_email: string (foreign key: users.email) (nullable)
organization_id: string (foreign key: organization.id)
location_id: string (foreign key: location.id)
sequence: number (minimum: 1)
type: enum ("collection", "distribution")
status: enum ("scheduled", "completed", "cancelled")
notes: string (nullable)
timestamp_created: date
timestamp_updated: date
timestamp_completed: date (nullable)
total_weight: number (minimum: 0) (nullable)
categorized_weight: {
	bakery: number (minimum: 0)
	dairy: number (minimum: 0)
	produce: number (minimum: 0)
	meat_fish: number (minimum: 0)
	non_perishable: number (minimum: 0)
	prepared_frozen: number (minimum: 0)
	mixed: number (minimum: 0)
	other: number (minimum: 0)
}
percent_of_total_dropped: number (minimum: 0) (maximum: 100) (nullable)
```

### Details

- **id**: The unique identifier for the transfer, which serves as the primary key.
- **rescue_id**: The ID of the rescue associated with the transfer, which is a foreign key referencing the `rescue.id` field.
- **handler_email**: The email of the handler responsible for the transfer, which is nullable and references the `users.email` field.
    - Note: if null, this is associated with an “available” rescue, meaning it is unassigned and can be claimed by a driver.
- **organization_id**: The ID of the organization associated with the transfer, which is a foreign key referencing the `organization.id` field.
- **location_id**: The ID of the location associated with the transfer, which is a foreign key referencing the `location.id` field.
- **sequence**: The sequence number of the transfer, which must be at least 1. This manages the order of transfers within a given rescue, which is of *particular* importance for retail rescues that can have intermingled collections and distributions.
- **type**: The type of the transfer, which can be one of the following values: `"collection"`, `"distribution"`.
    - “Collection” transfers are linked to donor organizations, and show food “coming in”.
    - “Distribution” transfers are linked to recipient organizations, and show food “going out”.
- **status**: The status of the transfer, which can be one of the following values: `"scheduled"`, `"completed"`, `"cancelled"`.
    - Note: we don’t include the “active” status that is available for the parent rescue. This led to misleading and inaccurate data based on driver’s ability to update their current status in realtime, and was best simplified.
- **notes**: Additional notes about the transfer, which are nullable.
- **timestamp_created**: The timestamp when the transfer was created.
- **timestamp_updated**: The timestamp when the transfer was last updated.
- **timestamp_completed**: The timestamp when the transfer was completed, which is nullable (note: this is required for transfers with status “completed”).
- **total_weight**: The total weight of the transfer, which must be at least 0 and is nullable (note: this is required for transfers with status “completed). This number must also equal the sum of all fields within “categorized_weight”.
- **categorized_weight**: An object showing the breakdown of weight between our 8 tracked food categories: `bakery`, `dairy`, `meat_fish`, `produce`, `prepared_frozen`, `non-perishable`, `mixed`, and `other`.
- **percent_of_total_dropped**: This is **only applicable for transfers of type “distribution”.** This is a number from 0 to 100 indicating the percentage of the available load that was dropped at this specific distribution. This could be classified as tech-debt, but is helpful in allowing the system to “recalculate” the “total_weight” and “categorized_weight” of a distribution should a previous transfer be edited.

## `user`

### Defintion

A user defines an individual person that uses the Sharing Excess tech platform. We use their email address as a primary identifier, and allow users to sign-in or sign-up by receiving one-time codes sent to that email address. We also store necessary insurance and liability information for drivers as a part of the user record, which is not made available on any public API endpoints.

### Properties

```
email: string (primary key)
organization_id: string (foreign key: organization.id) (nullable)
permission: enum (null, "read-only", "standard", "admin")
name: string
phone: string (nullable)
icon: string (nullable)
vehicle_make_model: string (nullable)
license_state: string (nullable)
license_number: string (nullable)
insurance_provider: string (nullable)
insurance_policy_number: string (nullable)
completed_liability_release: boolean (nullable)
timestamp_created: date
timestamp_updated: date
```

### Details

- **email**: The user's email address, which serves as the primary key and identifier throughout the application.
- **organization_id**: The ID of the organization the user is associated with, which is a foreign key referencing the `organization.id` field. This is used to associate partners with their specific organization, giving them access to their partner dashboard, without needing to provide the “admin” level access otherwise needed to access that data. This field is nullable.
- **permission**: The user's permission level, which can be one of the following values: `null`, `"standard"`, or `"admin"`. Those with `null` will not have any access throughout the app or partner dashboard. Those with `standard` have limited access, generally scoped by what’s absolutely necessary to be a retail driver. We also currently use `standard` access for partners, in which case an `organization_id` is necessary to provide dashboard access without any other meaningful access.
- **name**: The user's full name.
- **phone**: The user's phone number, which is nullable.
- **icon**: The URL to the user's icon or avatar image, which is nullable.
- **vehicle_make_model**: The make and model of the user's vehicle, which is nullable (note: this is necessary for retail drivers).
- **license_state**: The state of the user's driver's license, which is nullable (note: this is necessary for retail drivers).
- **license_number**: The user's driver's license number, which is nullable (note: this is necessary for retail drivers).
- **insurance_provider**: The user's insurance provider, which is nullable (note: this is necessary for retail drivers).
- **insurance_policy_number**: The user's insurance policy number, which is nullable and optional (note: this is necessary for retail drivers).
- **completed_liability_release**: A boolean indicating whether the user has agreed to the liability release within the app, which is nullable (note: this is necessary for retail drivers).
- **timestamp_created**: The timestamp when the user was created.
- **timestamp_updated**: The timestamp when the user was last updated.

## `organization`

### Defintion

An organization record defines an external organization with which Sharing Excess exchanges food. Organizations can be of type “donor” (meaning they donate food for us to distribute) or “recipient” (meaning we distribute donated food to them). An organization will be linked to one or more “locations”, which include specific address and contact information. This is importantly *separate* from the organization itself to allow for multiple locations under the umbrella of a single organization.

### Properties

```
id: string (primary key)
name: string
type: enum ("donor", "recipient")
subtype: enum ("food_bank", "agency", "community_fridge", "home_delivery", "popup", "compost", "holding", "other")
website: string (nullable)
timestamp_created: date
timestamp_updated: date
is_deleted: boolean (optional)
```

### Details

- **id**: The unique identifier for the organization, which serves as the primary key.
- **name**: The name of the organization.
- **type**: The high-level type of the organization, which can be one of the following values: `donor` , `recipient`. This determines whether the org can be used on transfers of type “collection” or “distribution”.
- **subtype**: The subtype of the organization, which can be one of the following values: `food_bank`, `agency`, `community_fridge`, `home_delivery`, `popup`, `compost`, `holding`, or `other`. Notably, organizations of subtype `holding` do *not* count toward’s total impact, as they’re defined as “intermediate” stops (ex. the SE Warehouse) and not end donors/recipients. Organizations of subtype `compost` are also counted uniquely, as we do not include their received food in our calculation of total “donated” weight (since this is not ultimately for human consumption).
- **website**: The website URL of the organization, which is nullable.
- **timestamp_created**: The timestamp when the organization was created.
- **timestamp_updated**: The timestamp when the organization was last updated.
- **is_deleted**: A boolean indicating whether the organization is deleted, which is optional.

## `location`

### Defintion

A location is an individual “brick and mortar” instance of a parent organization. An example would be the Market St. location of Trader Joe’s (which obviously has many additional locations). This record includes address information, latitude and longitude data for mapping and directions, and important contact information/driver instructions for SE staff to communicate with folks on the ground.

### Properties

```
id: string (primary key)
organization_id: string (foreign key: organization.id)
nickname: string (nullable)
contact_name: string (nullable)
contact_email: string (nullable)
contact_phone: string (nullable)
notes: string (nullable)
address1: string
address2: string (nullable)
city: string
state: string
zip: string
lat: number
lng: number
timestamp_created: date
timestamp_updated: date
is_deleted: boolean (optional)
```

### Details

- **email**: The user's email address, which serves as the primary key.
- **organization_id**: The ID of the organization the user is associated with, which is a foreign key referencing the `organization.id` field. This field is nullable.
- **permission**: The user's permission level, which can be one of the following values: `null`, `"read-only"`, `"standard"`, or `"admin"`.
- **name**: The user's name.
- **phone**: The user's phone number, which is nullable.
- **icon**: The URL or path to the user's icon or avatar, which is nullable.
- **vehicle_make_model**: The make and model of the user's vehicle, which is nullable and optional.
- **license_state**: The state of the user's driver's license, which is nullable and optional.
- **license_number**: The user's driver's license number, which is nullable and optional.
- **insurance_provider**: The user's insurance provider, which is nullable and optional.
- **insurance_policy_number**: The user's insurance policy number, which is nullable and optional.
- **completed_liability_release**: A boolean indicating whether the user has completed a liability release form, which is nullable and optional.
- **timestamp_created**: The timestamp when the user was created.
- **timestamp_updated**: The timestamp when the user was last updated.

## `location_tag`

### Defintion

Location tags are simple text-based tags that we use to group individual locations together. Example use cases are “PWPM Vendor” for donors inside the Philadelphia Wholesale Produce Market, or “PHLB Recipient” for recipients that are a part of the Philabundance approved network.

### Properties

```
id: string (primary key)
location_id: string (foreign key: location.id)
label: string
timestamp_created: date
```

### Details

- **id**: The unique identifier for the location tag, which serves as the primary key.
- **location_id**: The ID of the location associated with the tag, which is a foreign key referencing the `location.id` field.
- **label**: The label of the tag itself, which will be visible to the u.
- **timestamp_created**: The timestamp when the location tag was created.

## `wholesale_market`

### Defintion

A wholesale market is a very simple representation of one of the (currently 2) markets where we have physical space. Individual rescues can be labeled with a “wholesale_market_id”, which lets us know that the associated food did in fact move through the given wholesale market. 

Importantly, this allows us to report on the impact we create inside a given market (which can in fact go beyond the food donated from the market’s specific on-site vendors).

As a note: this could be potentially confusing with “location tags”. Many donor locations include either a “PWPM Vendor” or “Hunts Point Vendor” tag, and it’s also possible to use this as a filter when putting together impact reports. The difference: using “location tags” as a filter, will return specifically the food donated from vendors at a given market, while using the “wholesale market” filter will include food from rescues with the given market id (which likely includes all the food from on-site vendors, but could also come from an outside donor).

### Properties

```
id: string (primary key)
name: string
timestamp_created: date
timestamp_updated: date
```

### Details

- **id**: The unique identifier for the wholesale market, which serves as the primary key.
- **name**: The name of the wholesale market.
- **timestamp_created**: The timestamp when the wholesale market was created.
- **timestamp_updated**: The timestamp when the wholesale market was last updated.