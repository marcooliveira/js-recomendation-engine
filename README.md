# Recommendation engine

This recommendation engine framework is capable of analyzing and recommending entities to users based on the tastes of similar users and also based on the tastes of the user towards a set of features that you extract from entities.

Something to be kept in mind using this framework is that whenever a specific entity is being written and analyzed, it is guaranteed that the whole pipeline of that specific entity is isolated from any other operation, which means that "concurrent" writes of entities with the same id are serialized.

## 1. Introduction

**TODO: add simple example of instantiation and very simple usage.**

Throughout this documentation we'll be using a music recommendation engine as an example.

The engine is composed of:

- `Entities`: The items that the engine is capable of analyzing and recommending. In the music recommendation engine context, the entities would likely be the songs, albums and artists.
- `Users`: The users to which you're building the recommendation engine.
- `Entity-user relations`: Relations between a user and a specific entity. This could be the number of times that a user has played and commented on a song.

There are 2 very clear pipelines when writing into the engine, which ultimately affect the user's entity classification:

**Entity update:** `Write entity -> Entity Feature Extraction -> User Feature Extraction -> Generic classification -> User entity classification`

**Entity-user relation update:** `Write entity-user relation -> User Feature Extraction -> User entity classification`

### 1.1 Entities

*Entities* are composed of:

- `id`: The id of the entity.
- `data`: A copy of the provided entity data.
- `features`: An object with features and their respective values (using the music example, features could be an array of genres, the number of BPM).
- `classification`: An object with multiple classification entries, one per Generic Classifier (more on this below).

Here's how you can write (add or update) an entity into the engine:

```js
var songs = engine.entity('song');

songs.write({
    id: 'my_song_id',
    data: {
        plays:       124,
        likes:       33,
        shares:      19
        releaseDate: '2016-04-09',
        tags: [
            'electronic',
            'edm'
        ]
    }
}, function (err) {
    if (err) {
        return console.error(err);
    }
});
```

Note that `songs` is a `Transform` stream, so you can pipe into it and listen to all the events you'd expect from a `Transform` stream.

The `data` event provides the analyzed entity, as shown below.

```js
{
    id: 'my_song_id',
    data: {
        title: 'My song',
        plays: 124,
        likes: 33,
        shares: 19
        releaseDate: '2016-04-09',
        tags: [
            'electronic',
            'edm'
        ]
    },
    features: {
        plays: 124,
        likes: 33,
        shares: 19
        releaseDate: '2016-04-09',
        tags: [
            'electronic',
            'edm'
        ]
    },
    classification: {
        relevance: 247
        freshness: 1
    }
}
```

Below you'll see how features are extracted, using *Feature Extractors*, and how classifications are calculated, using *Generic and User Classifiers*.


### 1.2 Users

*Users* hold relations to both *Entities* and *Features*, and this relation has a numeric value associated (classification). Higher classification means a stronger relation between the two.

*Users* are composed of:

- `id`: The id of the user.
- `features`: An object with features and their respective values (using the music example, features could be an array of genres, the number of BPM).


### 1.3 Entity Feature Extractors

You can provide *Entity Feature Extractors*, per entity type, and they will be used internally to extract features from the entities along with their respective values, every time an entity is added or updated.

When called, the extractor is provided with:

- `entity`: The entity that was added or updated, which is an object containing:
    - `id`: The id of the entity.
    - `data`: The data of the new entity that was added or updated.
- `previousEntity`: An object with the previous entity before updating, giving you a chance to merge information, or `null` if the entity is being added for the first time. Contains the following properties:
    - `id`: The id of the entity.
    - `data`: The data of the previously added entity.
    - `features`: The previously extracted features of the entity.
- `callback(err, features)`: The callback when you're done after analyzing the entity:
    - `features`: An object with the features and respective values.

Here's an example:

```js
engine.entity('song').setFeatureExtractor(function (entity, previousEntity, callback) {
    // let's callback with the new features
    return callback(null, {
        plays: entity.data.plays,
        likes: entity.data.likes,
        shares: entity.data.shares,
        releaseDate: entity.data.releaseDate,
        tags: entity.data.tags
    });
});
```


### 1.4 Entity-user relations

You can describe relations between users and entities, which can be later used to calculate the user's classification of the entity.

```js
var userSongs = engine.entity('song').userRelations;

userSongs.write({
    entityId: 'my_song_id',
    userId:   'my_user_id',
    data: {
        liked:    true,
        comments: 2,
        shares:   1,
        plays:    13
    }
}, function (err) {
    if (err) {
        return console.error(err);
    }
});
```

Note that `userSongs` is a `Transform` stream, so you can pipe into it and listen to all the events you'd expect from a `Transform` stream.

The `data` event provides the already classified entity, along with the user and their relation:

```js
{
    entity: {
        id: 'my_song_id',
        data: {
            title: 'My song',
            plays: 124,
            likes: 33,
            shares: 19
            releaseDate: '2016-04-09',
            tags: [
                'electronic',
                'edm'
            ]
        },
        features: {
            plays: 124,
            likes: 33,
            shares: 19
            releaseDate: '2016-04-09',
            tags: [
                'electronic',
                'edm'
            ]
        },
        classification: {
            relevance: 247
            freshness: 1
        }
    },
    user: {
        id: 'my_user_id',
        features: {
            tags: {
                electronic: {
                    my_song_id: 24
                },
                edm: {
                    my_song_id: 24
                }
            }
        }
    },
    data: {
        liked:    true,
        comments: 2,
        shares:   1,
        plays:    13
    }
}
```

### 1.5 User Feature Extractors

You can provide *User Feature Extractors*, per entity type, and they will be used internally to extract user features along with their respective values, every time an entity or entity-user relation is added or updated.

When called, the extractor is provided with:

- `entity`: The entity that was added or updated, which is an object containing:
    - `id`: The id of the entity.
    - `data`: The data of the new entity that was added or updated.
    - `features`: The features extracted from the entity.
- `previousUser`:
    - `id`: The user id.
    - `features`: An object with the previous user features before updating, giving you a chance to merge information, or `null` if the user features are being extracted for the first time.
- `relation`: An object with the data of the relation between the user and entity.
- `callback(err, features)`: The callback when you're done after analyzing the entity:
    - `features`: An object with the features and respective values.

Here's an example:

```js
engine.entity('song').userRelations.setFeatureExtractor(function (entity, previousUser, relation, callback) {
    var features = previousUser.features || {
        tags: {}
    };

    // update the user's affinity for each song tag
    entity.features.tags.forEach(function (tag) {
        features.tags[tag][entity.id] = relation.shares * 3 + relation.comments * 3 + relation.plays + (relation.liked ? 2 : 0);
    });

    // let's callback with the new features
    return callback(null, features); // CONTINUE
});
```


### 1.6 Classifiers

Classifiers are functions provided by you, per entity type, and are internally used to compute the classification of an entity. There are *Generic Classifiers* and *User Classifiers*, more information on what each type does below.


#### 1.6.1 Generic Classifiers

*Generic Classifiers* are used to calculate a generic, user agnostic, classification of an entity, every time it is added or updated. You can add multiple *Generic Classifiers* per entity type and label them, in case you need to calculate multiple criteria.

Note that *Generic Classifiers* are called after the feature extractor has been called, so all the entity features are already extracted.

When called, the extractor is provided with:

- `id`: The id of the entity.
- `data`: The data of the new entity that was added or updated.
- `features`: The previously extracted features.
- `callback(err, classification)`: The callback when you're done after classifying the entity:
    - `classification`: A `Number` describing the generic classification of the entity. Higher classification is better.

In the example below, we're adding a couple of classifiers `relevance`, and `freshness` to the `song` entity:

```js
engine.entity('song').setGenericClassifier('relevance', function (entity, callback) {
    // shares weight 3x as much as a play, and likes weight 2x as much as a play
    return callback(null, (entity.features.plays * 1) + (entity.features.likes * 2) + (entity.features.shares * 3));
});

engine.entity('song').setGenericClassifier('freshness', function (entity, callback) {
    var daysSinceRelease = Math.round((Date.now() - (new Date(entity.data.releaseDate).getTime())) / 1000 / 60 / 60 / 24);

    // the more time goes by, the less "fresh" the song is
    return callback(null, 1 / Math.max(0, daysSinceRelease));
});
```


#### 1.6.2 User Classifiers

User Classifiers are functions provided by you, per entity type, and are internally used to calculate a user's classification of an entity, every time it is added or updated. Note that User Classifiers are called after the feature extractor has been called, and also after the Generic Classifiers, so all the entity features are already extracted and the generic classifications are available.

When called, the extractor is provided with:

- `userId`: The user who's classification of `entity` is being calculated.
- `entity`: The new entity that was added or updated.
- `features`: The extracted features.
- `callback(err, classification)`: The callback when you're done after classifying the entity:
    - `classification`: A `Number` describing the user's classification of the entity. Higher classification is better.

```js
engine.entity('song').setUserClassifier(function (user, entity, callback) {

});
```


## 2. Recommendations

### 2.1 User similarity driven

### 2.2 User-feature relation driven

## 3. Persistency

### 3.1 In-memory

### 3.2 Couchbase

### 3.3 Redis

### 3.4 ElasticSearch
