---
title: Callbacks and Refactoring
subheading: POROs and Callbacks o my!
---

## Goals

* Understand how callbacks work
* Know some common callbacks
* Start to understand how to implement a PORO
* Use callbacks to your advantage
* Use POROS to refactor controllers

## Repository

* `git clone -b starting_point https://github.com/Carmer/kitty_castle.git`
* We will start on the `starting_point` branch for this lesson

## Callbacks and POROs.
* This is our problem.

```ruby
class ReservationsController < ApplicationController
  def create
    credit_card = reservation_params[:credit_card_number]
    credit_card = credit_card.gsub(/-|\s/,'')
    reservation_params[:credit_card_number] = credit_card

    @reservation = Reservation.new(reservation_params)

    if @reservation.save
      flash[:notice] = "Reservation was created."
      ReservationMailer.reservation_confirmation(@reservation.kitty).deliver
      @reservation.kitty.update_attributes(status: “active”)
      redirect_to current_kitty
    else
      render :new
    end
  end

  private

  def reservation_params
    params.require(:reservation).permit(:credit_card_number, :kitty_id, :castle_id, :start_date, :end_date )
  end
end
  ```

  * This action is doing entirely too much. You're sanitizing the card number, sending an email if successful, updating the current_kitty.
  * This gets messy if we need to add additional behaviors. So we can refactor...

```ruby
class Reservation < ActiveRecord::Base
  before_validation :sanitize_credit_card
  after_create :send_reservation_confirmation
  after_save :set_kitty_to_active

  private

  def sanitize_credit_card
    credit_card.gsub(/-|\s/,'')
  end

  def send_reservation_confirmation
    ReservationMailer.reservation_confirmation(kitty).deliver
  end

  def set_kitty_to_active
    kitty.update_attributes(status: “active”)
  end
end
```

This refactor can be seen on the `refactor_controller` branch

* Meet `before_save` and `after_create`
  * What are callbacks?
  * Callbacks in real life? Take 3 minutes to brainstorm with a partner.


#### Rails/Active Record Callbacks:


1. Creating an Object
  * before_validation`
  * after_validation
  * before_save
  * around_save
  * before_create **__Note: before_create only gets called before a create.__**
  * around_create
  * after_create
  * after_save
  * after_commit/after_rollback
1. Updating an Object
  * before_validation
  * after_validation
  * before_save **__Note: before_save gets called when we update and when we create.__**
  * around_save
  * before_update
  * around_update
  * after_update
  * after_save
  * after_commit/after_rollback
1. Destroying an Object
  * before_destroy
  * around_destroy
  * after_destroy
  * after_commit/after_rollback

* These are some additional callbacks with their order of operations.






#### Good but not best...

* We just pulled a TON of things out of the controller.
* This is pretty good, but we can do better.
* The danger here is that the Reservation class knows *__entirely too much__* about other classes.
* This is dangerous because if you make a mistake somewhere, and say there's
a problem with something of the Kitty class, the Reservation class isn't really the first place a person would go look.

* If we keep this up, and we get a pretty unwieldy Reservation class that touches way too many other things and has too many responsibilities
* We should use a PORO instead.

```ruby
class ReservationsController < ApplicationController
  def create
    @reservation = Reservation.new(reservation_params)


    if @reservation.save
      ReservationCompletion.create(@reservation)
      flash[:notice] = "Reservation was created."
      redirect_to current_kitty
    else
      render :new
    end
  end

  private

  def reservation_params
    params.require(:reservation).permit(:credit_card_number, :kitty_id, :castle_id, :start_date, :end_date )
  end
end
```

```ruby
class ReservationCompletion

  def self.create(reservation)
     send_reservation_confirmation(reservation)
     set_kitty_to_active(reservation)
  end

  def self.send_reservation_confirmation(reservation)
    ReservationMailer.reservation_confirmation(reservation.kitty, reservation.castle).deliver
  end

  def self.set_kitty_to_active(reservation)
    reservation.kitty.update_attributes(status: “active”)
  end
end

```

This refactor can be seen on the `refactor_reservation_to_poro` branch.

* Here, we've moved all logic in reservation completion to a single place.
* You should only use a callback when it deals with the model instance you're currently working with.
* `after callbacks` are often code smells. That's why we fixed it.
* Callbacks that can trigger callbacks in other classes are Bad News [cat]Bears.

<!--
## Class Methods

* We can use class methods to do some filtering, and pushing logic down the
stack.
* We want to put the top three most expensive items in our index view.
* How can we get the information we need?
* Logic doesn't belong in the view.
* It doesn't belong in the controller either.
* There's one last place it can go. The model.

## Scopes

* Scopes allow you to define and chain query criteria in a declarative and
reusable manner.
* Scopes take lambdas.
* A lambda is a function without a name.
* We won't go into lambdas right now, but at the bottom of the page, there are resources where you can learn more about lambdas.
* Here's some examples.

```ruby
class Reservation < ActiveRecord::Base

  scope :complete, -> { where(complete: true) }
  scope :today, -> { where("start_date >= ?",
                           Time.zone.now.beginning_of_day) }
end
```

* They can take arguments.

```ruby
class Reservation < ActiveRecord::Base

    scope :newer_than, ->(date) {
        where("start_date > ?", date)
          }

end
```

* Let's convert our code into a scope.

## Scopes vs Class Methods
* These look eerily similar.
* But there are key differences.
* Scopes can always be chained.
* Class methods can be chained only if they return an object that can be chained.
* Scopes automatically work on has_many relationships.
* You can set up a default scope.


### Referring back to what we did

You can see all the work we did at github.com/carmer/kitty_castle on 5 different branches. `git clone https://github.com/Carmer/kitty_castle.git`

1. `git checkout starting_point` is our base starting point for this work
2. `git checkout refactor_controller` is our first iteration of refactoring the logic out of the controller
3. `git checkout refactor_reservation_to_poro` is our second iteration of refactoring logic our of the controller
4. `git checkout scopes` has our work of putting scopes into the project
5. `git checkout class_methods` has our work of putting class_methods into the project
 -->

## Other Resources:

* https://rubymonk.com/learning/books/1-ruby-primer/chapters/34-lambdas-and-blocks-in-ruby/lessons/77-lambdas-in-ruby
* https://rubymonk.com/learning/books/4-ruby-primer-ascent/chapters/18-blocks/lessons/64-blocks-procs-lambdas
* http://www.reactive.io/tips/2008/12/21/understanding-ruby-blocks-procs-and-lambdas/
* [Where to Put POROs](http://vrybas.github.io/blog/2014/08/15/a-way-to-organize-poros-in-rails/)
