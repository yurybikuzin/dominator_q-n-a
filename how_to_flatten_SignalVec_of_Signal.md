## How to flatten SignalVec of Signal?

I have following mod:
```
use dominator::clone;
use futures_signals::{
    signal::{Signal, SignalExt},
    signal_map::{MutableBTreeMap, SignalMapExt},
    signal_vec::{MutableVec, SignalVecExt},
};
use once_cell::sync::Lazy;
use std::sync::Arc;

pub static APP: Lazy<Arc<App>> = Lazy::new(App::new);

pub struct App {
    pub data: AppData,
}

#[derive(Default)]
pub struct AppData {
    pub cities_selected: MutableBTreeMap<u16, bool>,
    pub clubs: MutableVec<Arc<Club>>,
    pub clubs_map: MutableBTreeMap<u16, Arc<Club>>,
    pub judges: MutableVec<Arc<Judge>>,
}

pub struct City {
    pub id: u16,
    pub name: String,
}

pub struct Club {
    pub id: u16,
    pub city: u16,
}

pub struct Judge {
    pub club: u16,
}

pub struct Dancer {
    pub club: u16,
}

impl App {
    fn new() -> Arc<Self> {
        Arc::new(Self {
            data: AppData::default(),
        })
    }
    pub fn is_selected_city_of_club_of_the_dancer(dancer: Arc<Dancer>) -> impl Signal<Item = bool> {
        APP.data
            .clubs_map
            .signal_map_cloned()
            .key_cloned(dancer.club)
            .map(|club| {
                APP.data
                    .cities_selected
                    .signal_map_cloned()
                    .key_cloned(club.unwrap().city)
            })
            .flatten()
            .map(|is_selected| is_selected.unwrap_or(false))
    }
    pub fn sum_of_clubs_of_the_city(city: Arc<City>) -> impl Signal<Item = usize> {
        APP.data
            .clubs
            .signal_vec_cloned()
            .filter_map(clone!(city =>
               move |club|
                    if club.city == city.id {
                        Some(1)
                    } else {
                        None
                    }
            ))
            .sum()
    }
    pub fn sum_of_judges_from_clubs_of_the_city(city: Arc<City>) -> impl Signal<Item = usize> {
        APP.data
            .clubs
            .signal_vec_cloned()
            .filter_map(clone!(city => move |club|
                if club.city == city.id {
                    Some(
                        APP.data.judges.signal_vec_cloned()
                        .filter_map(clone!(club => move |judge|
                            (judge.club == club.id).then_some(1))
                        ).sum()
                    )
                } else {
                    None
                }
            ))
            .flatten()
            .sum()
    }
}
```
I'm getting:
```
92   |             .flatten()
     |              ^^^^^^^ method cannot be called on `futures_signals::signal_vec::FilterMap<MutableSignalVec<std::sync::Arc<to_pauan::Club>>, [closure@a100_front/src/to_pauan.rs:80:40: 90:18]>` due to unsatisfied trait bounds
```

The error concerns `.flatten()` in `fn sum_of_judges_from_clubs_of_the_city`, not `.flatten()` in `is_selected_city_of_club_of_the_dancer`

### Pauan's answer

the issue is that you're calling `flatten()`, but the types are wrong
`flatten()` exists on `Signal`, not `SignalVec`

so you probably want something like this instead:
```
    pub fn sum_of_judges_from_clubs_of_the_city(city: Arc<City>) -> impl Signal<Item = usize> {
        APP.data
            .clubs
            .signal_vec_cloned()
            .filter(clone!(city => move |club| club.city == city.id))
            .map_signal(move |club| {
                APP.data.judges.signal_vec_cloned()
                    .filter_map(clone!(club => move |judge|
                        (judge.club == club.id).then_some(1))
                    ).sum()
            })
            .sum()
    }
```

`map_signal` is the same as `map`, except the closure returns a `Signal` instead of a value
and it will then "flatten" that `Signal` into the `SignalVec`
so that means these two are the same:

```
some_signal_vec.map(|_| 5)

some_signal_vec.map_signal(|_| always(5))
```
 
they both return a `SignalVec<Item = i32>`
the difference between them is, with `map_signal`... when the `Signal` changes, it will automatically update the `SignalVec`

it is possible to convert a `SignalVec` into a `Signal`, by using the `to_signal_map` method:
```
some_signal_vec.to_signal_map(|elements| {
    ...
})
```
however, that's rather inefficient, since it will call that closure again every time any element in the `SignalVec` changes
using map_signal should be a lot more efficient
and then you don't need the `flatten()`


## Is there any consideration of choosing Arc over Rc in dominator app?

Is there any consideration of choosing Arc over Rc in dominator app? Or in the frontend use case?

For example of code see [Question "How to flatten SignalVec of Signal"](#how-to-flatten-signalvec-of-signal)

### Pauan's answer

I just always use Arc, in a single-threaded app there shouldn't be any performance difference, and in a multi-threaded app it allows you to share it between threads
but if you're using it in a single thread then Rc is perfectly fine too

but if you want to store it in a static then you have to use Arc
so if you're using Rc then you have to use thread_local! instead

