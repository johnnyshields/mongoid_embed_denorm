# WORK IN PROGRESS

## Mongoid::ClassyDenorm

**Mongoid Denormalization via Embedded Models**

Stay classy with an object-oriented approach to Mongoid denormalization.

![Stay Classy](https://cloud.githubusercontent.com/assets/27655/3567255/3d2382f4-0b14-11e4-87f7-954e7fd35ecb.jpg)

Mongoid::ClassyDenorm defines model callback actions which copy denormalized data from a source document to an embedded
"container" document within a target document. Pull (saving target "pulls" from source), Push (saving source "pushes" to
target), and Bi-directional modes are supported. Unlike other Mongoid denormalization gems, Mongoid::ClassyDenorm does
**NOT** auto-define models/data-structures/relations used to contain the denormalized data. This task is left to the
implementor for maximum flexibility; a worked example is provided below.

Mongoid::ClassyDenorm is inspired by [@dzello](https://github.com/dzello)'s fantastic [Mongoid::Alize](https://github.com/dzello/mongoid_alize) gem.


### Usage

* **Step 1)** Choose a document (the *"Source"*) to denormalize into another document (the *"Target"*).

* **Step 2)** Create a new embedded document to contain the denormalized data (the *"Container"*):
    * The *Container* SHOULD have a subset of the fields of the *Source*. Please ensure the field names and types are identical.
    * The *Container* MUST be `embedded_in` the *Target*.
    * The *Container* MUST `belongs_to` the *Source*.

* **Step 3)** Specify **one** of the following three `classy_denorm` macro patterns to setup callbacks:

   * In your *Target* document, you can specify either bi-directional mode **OR** `:pull` mode:

   ```ruby
   # in "Target" document - Bi-directional push/pull sync
   classy_denorm <source_relation>, <container_relation>
   ```

   ```ruby
   # in "Target" document - One-way sync via pull
   classy_denorm <source_relation>, <container_relation>, only: :pull
   ```

   * **OR** in your *Source* document, you specify use `:push` mode:

   ```ruby
   # in "Source" document - One-way sync via push
   classy_denorm <target_relation>, <container_relation>, only: :push
   ```


### Worked Example

* In this example, we wish to denormalize `Appointment` (the *source*) to `Customer` (the *target*), using `DenormAppointment` as the *container*.

```ruby
# the Source
class Appointment
  include Mongoid::Document

  field :date,    type: Time
  field :status,  type: Symbol
  field :reason,  type: String
  # other fields...

  belongs_to :customer
end


# the "Target"
class Customer
  include Mongoid::Document
  include Mongoid::ClassyDenorm

  # REQUIRED: Define the relation you wish to denormalize
  has_many :appointments

  # REQUIRED: Define the embedded relation to the denormalization target
  embeds_many :denorm_appointments do

     # TIP: adding extensions to denormalized data (via a block) can be especially useful
     # to reduce query traffic. See: http://mongoid.org/en/mongoid/docs/relations.html
     def late_count
       @target.select{ |a| a.late? }.size
     end
  end

  # REQUIRED: Call the `classy_denorm` macro
  classy_denorm :appointments, :denorm_appointments
end


# the "Container"
class DenormAppointment
  include Mongoid::Document

  # REQUIRED: define an `embedded_in` to the "Target"
  embedded_in :customer

  # REQUIRED: Define a `has_one` relation to the "Source"
  belong_to :appointment

  # ClassyDenorm will copy values to fields whose names match those of the Source document
  field :date,    type: Time
  field :status,  type: Symbol

  # Fields whose names are not present on the Source document will be ignored
  field :unused,  type: String

  # TIP: You may define custom methods on your denormalized objects. Consider using a mixin
  # to support the same methods on both the original and denormalized models.
  def late?
    status == :late
  end
end


# Usage Examples

customer    = Customer.create
appointment = Appointment.create(date: Date.today, status: :late, reason: "Apply for fish license")

# Create
customer.appointments << appointment      # auto-creates DenormAppointment record
d_appointment = customer.reload.denorm_appointments.first
d_appointment.appointment == appointment  #=> true
d_appointment.late?                       #=> true
customer.denorm_appointments.late_count   #=> 1

# Update
appointment.update_attributes(status: :on_time)   # auto-updates DenormAppointment record
customer.reload.denorm_appointments.first.late?   #=> false

# Destroy
customer.appointments.clear               # auto-destroys DenormAppointment record
customer.reload.denorm_appointments       #=> []
```


### Comparison with Mongoid::Alize

[Mongoid::Alize](https://github.com/dzello/mongoid_alize) supports the full gamut of denormalization methods (bidirectional/push/pull).
However, Alize persists denormalized data as a `Hash` (or `Array<Hash>`). This has the following drawbacks:

* One cannot code methods into the denormalized `Hash` objects as if they were regular models.

* Mongoid's default demongoization is used on the denormalized `Hash`, which does not preserve `DateTime` vs. `Date`, `String` vs. `Symbol`, etc. due to ambiguities between mapping MongoDB types to Ruby types [(see here)](https://github.com/dzello/mongoid_alize/issues/18).

* Parts of the Alize gem code are in practice duplicating Mongoid's built-in embedded document functionality. This adds extra complexity and testing overhead to the gem.

While I contributed several PRs to Alizé, ultimately I felt that a clean-slate approach was warranted to use embedded documents.
