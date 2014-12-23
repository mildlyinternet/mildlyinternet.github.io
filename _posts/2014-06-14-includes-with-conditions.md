---
layout: post
title: Beware Includes with Conditions
---

##N+1 Problem

Lets say that you want to print out all of the invoices for each of your
cusomters. Your code may look like this:

{% highlight erb %}
<% @customers.each do |customer| %>
  <%= customer.name %>
  <% customer.invoices.each do |invoice| %>
      <%= invoice.details %>
  <% end %>
<% end %>
{% endhighlight %}

For every loop in customers, we have to go back to the database to fetch the
invoices associated with that customer. This is quite slow, and will lead to
performance problems. It is known as the *N+1* problem - you will issue `N+1`
queries, where N is the number of customers. If you look at the server log, you
will see multiple queries being run.

{% highlight sql %}
Invoice Load (0.2ms)  SELECT "invoices".* FROM "invoices"  WHERE "invouces"."customer_id" = $1  [["customer_id", 1]]
Invoice Load (0.2ms)  SELECT "invoices".* FROM "invoices"  WHERE "invouces"."customer_id" = $1  [["customer_id", 2]]
Invoice Load (0.2ms)  SELECT "invoices".* FROM "invoices"  WHERE "invouces"."customer_id" = $1  [["customer_id", 3]]
...
...
{% endhighlight %}

##Using Includes

A simple solution to this problem is to simple preload all of the invoices for the cusomters, using an `includes` statement.

{% highlight ruby %}
@customers = Customer.all.includes(:invoices)
{% endhighlight %}

Now you just generate two small, tight sql querires. 

{% highlight sql %}
Customer Load (1.1ms)  SELECT "customers".* FROM "customers"
Invoice Load (0.4ms)  SELECT "invoices".* FROM "invoices"  WHERE "invoices"."customer_id" IN (1, 2, 3, 4, 5, 6)
{% endhighlight %}

##Getting into trouble
Now lets say we only want to show the invoices that are have a status of "Open". We may be tempted to add a `where` condition on our include to do the filtering for us.

{% highlight ruby %}
@customers = Customer.all.includes(:invoices).where(invoices: { status: "open" })
{% endhighlight %}

Unfortunately, Rails cannot figure out how to optimize and break apart this query, so it will **Left Outer Join** everything...

Here's output from a different set of models.

{% highlight ruby %}
User.includes(:memberships).where(memberships: { status: "active" })

SQL (1.0ms)  SELECT "users"."id" AS t0_r0, "users"."avatar_url" AS t0_r1,
"users"."email" AS t0_r2, "users"."first_name" AS t0_r3, "users"."last_name" AS
t0_r4, "users"."oauth_provider" AS t0_r5, "users"."oauth_access_token" AS
t0_r6, "users"."oauth_remember_token" AS t0_r7, "users"."oauth_uid" AS t0_r8,
"users"."password_digest" AS t0_r9, "users"."password_reset_token" AS t0_r10,
"users"."remember_token" AS t0_r11, "users"."last_digest_sent_at" AS t0_r12,
"users"."password_reset_token_expires_at" AS t0_r13, "users"."created_at" AS
t0_r14, "users"."updated_at" AS t0_r15, "users"."users_invited" AS t0_r16,
"users"."phone_number" AS t0_r17, "users"."company" AS t0_r18,
"memberships"."id" AS t1_r0, "memberships"."project_id" AS t1_r1,
"memberships"."user_id" AS t1_r2, "memberships"."status" AS t1_r3,
"memberships"."is_admin" AS t1_r4, "memberships"."last_viewed_events_at" AS
t1_r5, "memberships"."created_at" AS t1_r6, "memberships"."updated_at" AS t1_r7
FROM "users" LEFT OUTER JOIN "memberships" ON "memberships"."user_id" =
"users"."id" WHERE "memberships"."status" = 'active'
{% endhighlight %}

The query above, while large, it is still much more performant than dealing
with N+1 problems. However, once the tables start to get large or you are
returning a large amount of rows, the query like the one above can be
potentially troublesome. The database returns something like this:

```
customer_id |   name  | invoice_id | title
------------|---------|------------|----------------------------
1           | malcolm | 1          | Some invoice title
1           | malcolm | 2          | Another invoice
2           | wash    | 3          | Late invoice title
3           | zoe     | 4          | Some invoice title
3           | zoe     | 5          | Another title
3           | zoe     | 6          | Invoice Title
```

We get some duplication - a customer has_many invoices so every time we load a
unique invoice we are going to load ALL of the customer data along with it. 

##Double the `has_many`

Now, issues arise when you try and eager load data from more than one has_many.

{% highlight ruby %}
@customers = Customer.all.includes(:invoices, :receipts).where(invoices: { status: "open" })
{% endhighlight %}

This generates two left outer joins. Whats going to happen is that the database
is going to return a row for every unique combination of customer, invoice and
receipt. So if we have 25 customers, each customer has 10 invoices (250 total)
and 10 receipts (250 total). The result set that is returned is not 525
records, but something closer to 2500 records. Now Rails has to instantiate all
of that into ActiveRecord objects.

##When is it safe to add where conditions with an includes?

Only when conditions are used on the main model

{% highlight ruby %}
Customer.all.includes(:invoices, :receipts).where("users.name" => "patkoperwas")
{% endhighlight %}

This will generate just 3 small queries.

{% highlight sql %}
Customer Load (19.4ms)  SELECT `customers`.* FROM `customers` WHERE `users`.`name` = 'patkoperwas'
Invoice Load (14.7ms)  SELECT `invoices`.* FROM `invoices` WHERE `invoices`.`customer_id` IN (1) ORDER BY invoices.created_at desc
Receipt Load (5.0ms)  SELECT `receipts `.* FROM `receipts` WHERE `receipts`.`customer_id` IN (1) ORDER BY receipts.id DESC
{% endhighlight %}
