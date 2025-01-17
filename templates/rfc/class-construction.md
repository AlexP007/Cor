# Overview

For object construction, we provide a list of needed steps and then show
pseudocode to make the construction process explicit.

**Note**: because Corinna is single inheritance, MRO order is simply child to
parent.

Anything which can be removed from this will make object construction
faster. Anything which can be pushed to compile-time will make object
construction faster.

This is almost certainly incorrect, but it's a start.

Also, roles get an ADJUST phaser now

1. Check that the even-sized list of args to new() are not duplicated
   (stops the new( this => 1, this => 2 ) error)
2. And that the keys are not references
3. Walk through classes in reverse MRO order. Croak() if any attribute
   name is reused
4. After previous step, if we have any extra keys passed to new() which cannot be
   allocated to a slot, throw an exception
5. For the internal NEW phaser, assign all values to their correct slots in
   reverse mro order
6. Call all ADJUST phasers in reverse MRO order (no need to validate here because
   everything should be checked at this point)

# Steps

## Step 1 Verify even-sized list of args

Check that the even-sized list of args to `new()` are not duplicated (stops
the `new( this => 1, this => 2` ) error).

```perl
my @args = ... get list passed to new()
  unless ( !( @args % 2 ) ) {
    croak("even-sized list required");
}
```

## Step 2 Constructor keys may not be references

Keys are not references (duh).

```perl
my %arg_for;
while (@args) {
    my ( $key, $value ) = splice @args, 0, 2;
    if ( ref $key ) {
        croak("$key must not be a ref");
    }
    if ( exists $arg_for{$key} ) {
        croak("duplicate key $key detected");
    }
    $arg_for{$key} = $value;
}
```

## Step 3 Find constructor args

Walk through classes from parent to child. `croak()` if any constructor
argument is reused.

```perl
my %orig_args = %arg_for;    # shallow copy
my %constructor_args;


my @duplicate_constructor_args;
foreach my $class (@reverse_mro) {
    my @roles = roles_from_class($class);
    foreach my $thing ( $class, @roles ) {
        foreach my $name ( get_slots_with_new_attribute($thing) ) {
            if ( my $other_class = $constructor_args{$name} ) {
                # XXX Warning! This may be a bad thing
                # If you don't happen to notice that some parent class has done
                # `slot $cutoff :param = 42;`
                # then you might accidentally write:
                # `slot $cutoff :param = DateTime->now->add(days => 7);`
                # instead, we probably need some way of signaling this to the
                # programmer. A compile-time error would be good.
                push @duplicate_constructor_args 
                  => "Arg $name in $thing already used in $other_class";
            }
            $constructor_args{$name} = $class;
        }
    }
}
if (my $error = join '  ' => @duplicate_constructor_args) {
    croak($error);
}
```

## Step 4 Err out on unknown keys


After previous step, if we have any extra keys passed to `new()` which cannot
be allocated to a slot, throw an exception this works because by the time we
get to the final class, all keys should be accounted for. Stops the issue of
`Class->new(feild => 4)` when the slot is `slot $field :param = 3;`

```perl
my @bad_keys;
foreach my $key ( keys %arg_for ) {
    push @bad_keys => $key unless exists $constructor_args{$key};
}
if (@bad_keys) {
    croak(...);
}
```

## Step 5 `new()`

For the internal NEW phaser, assign all values to their correct slots from
parent to child.

```perl
my @slot_values;
foreach my $this_class (@reverse_mro) {
    my @roles = roles_from_class($class);
    foreach my $thing ( $class, @roles ) {
        foreach my $slot_name ( get_slots_in_initialization_order($thing) ) {
            push @slot_values => $arg_for{$slot_name};
        }
    }
}
my $self = bless \@slot_values => $class;
```

## Step 6 `ADJUST`

Call all `ADJUST` phasers from parent to childre (no need to validate here because
everything should be checked at this point).

```perl
foreach my $class (@reverse_mro) {
    my @roles = roles_from_class($class);
    foreach my $thing ( $class, @roles ) {
        $thing::ADJUST ( $self, %arg_for );    # phaser, not a method
    }
}
```

# MOP Pseudocode

MOP stuff

```perl
class MOP {
    method get_slots_with_new_attributes($class_or_role) {
        return
          grep { $self->has_attribute( 'new', $_ ) }
          get_all_slots($class_or_role);
    }

    method get_slots_in_initialization_order($class_or_role) {
        # get_all_slots($class_or_role) should return them in declaration order
        my @slots = get_all_slots($class_or_role);
        my @ordered;
        my $constructor_args_processed = 0;
        while (@slots) {
            my $slot = shift @slots;
            if ( $self->has_attribute( 'new', $slot ) ) {
                push @ordered => $slots;
                my @remaining;
                foreach my $slot (@slots) {
                    if ( $self->has_attribute( 'new', $slot ) ) {
                        push @ordered => $slot;
                    }
                    else {
                        push @remaining => $slot;
                    }
                }
                @slots = @remaining;
            }
            else {
                push @ordered => $slot;
            }
        }
    }
}
```
