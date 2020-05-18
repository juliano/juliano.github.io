---
layout: single
title: Picking the best tool
header:
  teaser: /assets/images/washing-hands.jpg
  imgcredit: Photo by @burst from Pexels
categories:
  - bash
  - scripting
---

The best tool is often the simplest, isn't it? Still, doesn't matter how experienced one can be, once in a while passion overcomes logic.

## The Problem

As mentioned before, [I've been writing an API client](/2020/04/20/zio-http4s-a-simple-api-client/) to consume public data. The idea is reshaping the data and expose it in a useful way.

[Turns out the API](http://www.transparencia.gov.br/swagger-ui.html){:target="_blank"} is not as good as I was expecting. Timeouts, inconsistencies and corner cases make it unreliable. At least there's an alternative, [I can download the data I need in CSV files](http://portaltransparencia.gov.br/download-de-dados){:target="_blank"}.

## The Plan

Let's have a look at what should be done:

- download a zip file
- unzip it, the content will be the CSV file
- change the encoding from ISO-8859-1 to UTF-8
- transform it in json
- upload the resulting json to Amazon S3

## The Passionate Solution

I started building the solution with [ZStream](https://zio.dev/docs/datatypes/datatypes_stream){:target="_blank"}. I mean, if [one can build a Space Station with ZStream](/2020/05/04/deathstar-zio-stream/), it would be easy to solve such a simpler task using it as well.

Downloaded the byte stream, unzipped it... but I couldn't parse the content to *UTF-8* because the file is actually encoded in *ISO-8859-1*.

> Maybe I could implement that myself? That would be fun! It's very low level... could use the opportunity to write some Rust instead!

### Stop right there!

![](/assets/images/stop.png){: .align-center}

Immediately, the picture of Tim Urban poped up in my mind:

<iframe width="560" height="315" src="https://www.youtube.com/embed/arj7oStGLkU" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

> the excitement of working with newer/nicer technology can affect our decisions, specialy when it's a personal project and we don't have deadlines.

I would probably have hours and hours of fun, researching and implementing a `ZSink.iso88591Decode` or finding the right Rust crate to play with.

I did neither. And still, I had a lot of fun! After getting back to reality, the answer was clear: bash tools were ideal to do the job.

## The Logic (and smart) Solution

### Downloading a zip file

[curl](https://curl.haxx.se/){:target="_blank"} is the obvious solution:

```bash
_download_zip() {
  echo "Downloading zip file from ${URL}"
  curl -f ${URL} -o ${ZIPFILE}
}
```

### Unzipping and changing csv encoding

`unzip` is trivial, but changing file encoding with [iconv](https://www.gnu.org/savannah-checkouts/gnu/libiconv/documentation/libiconv-1.15/iconv.1.html){:target="_blank"} was new to me:

```bash
_unzip_to_utf8() {
  echo "Unzipping ${ZIPFILE} and converting content encoding to UTF-8"
  unzip -p ${ZIPFILE} | iconv -f ISO-8859-1 -t UTF-8 > ${CSVFILE}
}
```

I could use two separate functions, but `unzip` is way to simple. Besides, piping the two commands saves the space of an intermediate file :)

### CSV to JSON

I am familiar with [jq](https://stedolan.github.io/jq/){:target="_blank"}, but using it to handle CSV is new to me:

```bash
_csv_to_json() {
  echo "Converting ${CSVFILE} to JSON"
  jq -Rsfc csv2json.jq ${CSVFILE} > ${JSONFILE}
}
```

See the `csv2json.jq` file? Coding it was the complex (and fun!) part - I had no idea [`jq` has it's own programming language!](https://github.com/stedolan/jq/wiki/jq-Language-Description#the-jq-language){:target="_blank"}

```javascript
def objectify(headers):
  def tonumberq: tonumber? // .;
  . as $in
  | reduce range(0; headers|length) as $i ({}; .[headers[$i]] = ($in[$i] | tonumberq) );

def csv2table:
  # For jq 1.4, replace the following line by:  def trim: .;
  def stripr: sub("\r+$";"");
  def trim: sub("^ +";"") |  sub(" +$";"");
  def noquotes: sub("^\"+";"") |  sub("\"+$";"");
  split("\n") | map(stripr) | map( split(";") | map(trim) | map(noquotes) );

def csv2json:
  csv2table
  | .[0] as $headers
  | reduce (.[1:][] | select(length > 0) ) as $row
      ( []; . + [ $row|objectify($headers) ]);

csv2json
```

The `csv2table` function splits the input string on the separator `\n`, creating an array. All the other functions defined under `csv2table` are there to clean up the content.

`csv2json` takes the array, uses the first element as `$headers` and then `reduce` the rest of the array using `objectify`, which basically assembles every json object.

In summary, the script transforms this:

|id|department|
|----|-------|
|42|Hitchhikers|
|13|Unlucky|

into that:

```json
[
  {
    "id": 42,
    "department": "Hitchhikers"
  },
  {
    "id": 13,
    "department": "Unlucky"
  }
]
```

In a functional way! How cool is that?

### Uploading to S3

Another easy step, [making use of `aws-cli`](https://aws.amazon.com/cli/){:target="_blank"}:

```bash
_upload_to_s3() {
  local s3_url="s3://${BUCKET}/${FOLDER}/"
  echo "Uploading file to S3"
  aws s3 cp ${JSONFILE} ${s3_url}
}
```

### Final steps

Hygiene is a must:

```bash
_cleanup() {
  echo "Cleaning up"
  rm -r ${ZIPFILE} ${CSVFILE} ${JSONFILE}
}
```

The final result is that simple:

```bash
_download_zip && _unzip_to_utf8 && _csv_to_json && _upload_to_s3
trap _cleanup EXIT
```

## Conclusion

The best tool to do the job isn't the newest or the most interesting, many times it's the simplest. It's hard to define what *"the simplest mean"*, so I would say the main aspects to be considered are:

- the return on **time invested**
- the **environment** where the solution will live - and grow

Thinking about using the minimum amount of time, using existing tools, that I already know, is a better approach than coding from scratch. However, it's easy to trick myself here: because there are no deadlines, one could argue that the time is being invested in the knowledge acquired while studying the new tool - as I did - what's the perfect excuse to lose focus.

The environment where it will live is equally important; in my case the bash script solution is ideal because it won't grow, it won't get more complex, the problem is now solved. If it wasn't the case, can you imagine writing a whole ecosystem in bash?

Something that works well for me that I'd like to share, regarding personal projects: don't work all by yourself if you can. Having someone depending on the outcome of your work helps a lot to keep focus. Invite a friend to code with you or to play the Product Owner, that will help to keep attention on what actually matters.
