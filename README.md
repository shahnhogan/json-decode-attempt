# MackeyRMS/json-decode-attempt

I have had some ideas about JSON decoding maturing in my head for the last few years, largely in reflection of the decoders I have had to write at work. These ideas have finally culminated in a package I am really happy to share with you all called `MackeyRMS/json-decode-attempt`. In this post, I will talk about the ideas first, and then share the code at the end, but if you want to skip the words and see the package you can [click here](https://package.elm-lang.org/packages/MackeyRMS/json-decode-attempt/latest).

## A spectrum of decode failures
If you have done any JSON decoding at all in Elm, then you know that JSON decoders can fail. Furthermore, your Elm program needs to know what to do when a JSON decoder fails. In some way or another, that means using a fallback value instead. That fallback value can be anywhere on a spectrum of severity. The least severe response would be a simple default value (like, if you can't decode a list, just use `[]` instead), and the most severe response would be switching your entire `Model` into some failure mode (such as, if you can't decode a list, cancel everything and write “SORRY” across the screen in big letters). Both of these are fallbacks in the sense that we settle with our next-best user experience given the decoding failure. The art of decoding seems to be picking where along that spectrum of failure is the best user experience.

Figuring out the best spot on that spectrum depends quite a lot on what it is you are decoding. Consider this `User` type for example, which only has an id and a profile picture.
```elm
type alias User =
    { id : String
    , profilePic : String
    }
```
A profile picture failing to decode is probably on the less severe side of the fallback spectrum I mentioned above. We can fall back to a default profile picture and still provide 90% of the positive UX. Not getting a user's profile picture is not a good outcome, but it is an outcome we can recover from. On the other hand, a user id failing to decode is much closer to a catastrophe, because without the id there are many fundamental things you can't do regarding that user (such as updating the user or requesting more information about the user). There is no valid default value for a particular user id because if you fake the user id you are going to cause big problems elsewhere (like when you try and use a fake user id).

The bottom line is, just in terms of UX value, not all JSON decode failures are equal.

Now given that, consider this straightforward decoder for a `User`
```elm
decoder : Decoder User
decoder =
    Decode.map2 User
        (Decode.field "id" Decode.string)
        (Decode.field "profilePic" Decode.string)
```
Despite the `id` being much more mission-critical than a `profilePic` they both can cause the entire `Decoder User` to fail in a big way. If either fails, the whole thing fails, and if the whole thing fails, we don't get either field. Failing to decode the `profilePic` is just as bad as failing to decode the `id` because both failures lead to not getting the `id`. Generalizing a bit; every additional field you want to decode increases the probability that at least one field will fail, which in turn increases the probability that all of them ultimately fail. Once your data grows to 7 or more fields in a record, decoders start to feel like ticking time bombs in your application.

## Just fall back
There is an easy solution to all this, which is to just fall back to default values where you can. Here is a `User` decoder where the profile picture falls back.
```elm
decoder : Decoder User
decoder =
    Decode.map2 User
        (Decode.field "id" Decode.string)
        (Decode.oneOf
            [ Decode.field "profilePic" Decode.string
            , Decode.succeed defaultProfilePic
            ]
        )
```
Easy. By falling back we have done a lot to reduce this field interdependency problem. If the `profilePic` field fails, it simply falls back to the default profile picture and we don't lose the `id` as well.

## But there is a new problem: errors are silent
One thing I love about decode failures is that you get a `Decoder.Error` as an output. From your `Decode.Error` you can see exactly what went wrong and then fix it. In my own work, I try and route `Decode.Error` into a remote logging service so I know when my company's customers experience these problems (after scrubbing the error of sensitive information).

One thing I _don't_ like about using fallback values in decoders is that I lose the errors. It's a double-edged sword. With a fallback value, the decoder technically doesn't fail, but it doesn't truly succeed either, so I lose the precious `Decode.Error` information. In the decoder above, I can never get a `Decoder.Error` pertaining to why it failed to decode the profile picture, even though it did in fact fail.

 Falling back silently tends to prolong the lifetime of these bugs because they go undetected. When the `Decode.`Error` doesn't exist, `the problem `doesn't` end up` in my analytics pipeline. Without automated reporting, we only have the next-best forms of error reporting such as rumors from someone that it "didn't work" for who knows why. Hopefully, you get better user feedback than that, but at least you can see that the odds of getting delayed and insufficient user feedback on these kinds of errors greatly increase when there isn't a `Decode.Error` to work with.

## Decoding Loudly and Gracefully
With Elm decoders as they are, there is a trade-off between graceful error handling and being able to observe errors at all. **Either you can fail hard (bad) and loud (good), or you can fail gracefully (good) but quietly (bad), but you can't get the good of both (loud and graceful).**

Here is where `MackeyRMS/json-decode-attempt` comes in.
```elm
import Json.Decode.Attempt as Attempt exposing (Attempt, Safely, Dangerously)
import Json.Decode as Decode

Attempt.decodeValueSafely : Attempt value Safely -> Decode.Value -> (value, List Decode.Error)
```

With this API, if you decode things in a 100% safe way where everything has a fallback, then it's actually impossible for it to fail. The type signature reflects this; it's not a `Result Decode.Error value` like with regular JSON decoding, it's a `(value, List Decode.Error)`, so you _always always_ get the `value`. Furthermore, you get what you want without suppressing any errors. The API still returns a `List Decode.Error` representing- not just one- but _every_ error it encountered along the way.

So how do you make one of these `Attempt value safety` anyway? It's just like the classic `NoRedInk/json-decode-pipeline` package.

```elm
import Json.Decode.Attempt as Attempt exposing (Attempt, Safely, Dangerously)

type alias Image =
    { dataUrl : String
    , createdAt : Maybe Time.Posix
    }

userDecoder : Attempt Image Safely
userDecoder =
    Attempt.fromValue Image
        |> Attempt.try (Decode.field "dataUrl" Decode.string) dataMissingImage
        |> Attempt.try
            (Decode.field "createdAt" Decode.int
                |> Decode.map Time.millisToPosix
                |> Decode.map Just
            )
            Nothing
```
The `Safely` type is important. It is kind of like a promise that there is a fallback for every step and therefore the attempt cannot possibly fail. It therefore meets the type requirements to be decoded using `Attempt.decodeValueSafely`.

What about `Dangerously`?

```elm
import Json.Decode.Attempt as Attempt exposing (Attempt, Safely, Dangerously)

type alias User =
    { id : String
    , profilePic : String
    }

userDecoder : Attempt User Dangerously
userDecoder =
    Attempt.fromValue User
        |> Attempt.risk (Decode.field "id" Decode.string)
        |> Attempt.try (Decode.field "profilePic" Decode.string) defaultProfilePic
```
There's no one-size-fits-all solution; you shouldn't always fall back. Failing to decode the user id is a big problem, and it's better to simply fail in a big and loud way that doesn't even try and fall back. In the code above, this decoder uses `Attempt.risk` for the id, which contaminates the entire `Attempt`, transforming it from a `Safely` to a `Dangerously`. It is now possible for the entire process to fail so you _must_ use `Attempt.decodeValueDangerously`, and you _must_ handle the case that one of these critical errors occurs:
```elm
import Json.Decode.Attempt as Attempt exposing (Attempt, Safely, Dangerously)
import Json.Decode as Decode

Attempt.decodeValueDangerously :
    Attempt value Dangerously
    -> Decode.Value
    -> Result Decode.Error ( value, List Decode.Error )
```
Now we have a `Result Decoder.Error ..` again and you do have to handle an `Err _` case for the big critical failures. But even here, we still have an advantage over regular JSON decoding. The `Ok` case contains all the minor errors we accumulated along the way.