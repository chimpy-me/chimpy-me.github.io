---
title: What the Birthday Paradox Teaches Us About Protecting Patron Privacy
date: 2023-09-26
categories: [Library Data]
tags: [patrons, python, privacy]     # TAG names should always be lowercase
image:
  path: /_preview_a_very_chimpy_unbirthday.webp
  alt: A Very Chimpy Unbirthday
math: true
author: <ray_voelker>
pin: true
---

**Update 2023-10-01** The implementation shown below is now available to install via [Python PIP](https://pypi.org/project/stochastic-pseudonymizer/): 

```bash
python3 -m pip install stochastic-pseudonymizer
```

View the source at [GitHub](https://github.com/chimpy-me/stochastic-pseudonymizer) and 
discuss on [HackerNews](https://news.ycombinator.com/item?id=37696509)

----

## Background

I'm the Integrated Library System (ILS) Administrator for a [large public library system in Ohio](https://chpl.org). Libraries often struggle with data—being especially sensitive around data related to patrons and patron behavior in terms of borrowing, library program attendance, reference questions, etc. The common practice is for libraries to **aggregate and then promptly destroy this data** within a short time frame—which is typically one month. However, administrators and local government officials, who are often instrumental in allocating library funding and guiding operational strategies, frequently ask questions on a larger time scale than one month to validate the library's significance and its operational strategies. **Disaggregating this data to answer such questions is very difficult and arguably *impossible***. This puts people like me, and many others like me, in a tough spot in terms of storing and later using sensitive data to provide the answers to these questions of pretty serious consequence—like, what should we spend money on, or why we should continue to exist.

![banned books](/banned-books.jpeg "read banned books"){: .w-50 .right width="1024" height="768"}

I’m sure many readers are aware of the [many](https://en.wikipedia.org/wiki/McCarthyism) [interesting](https://en.wikipedia.org/wiki/National_security_letter#Connecticut_librarians_case) [historical](https://en.wikipedia.org/wiki/Patriot_Act) [reasons](https://en.wikipedia.org/wiki/Book_censorship_in_the_United_States) for this sensitivity, and organizations like the American Library Association (ALA) and other international library associations have even codified the protection of patron privacy into their codes of ethics. For example, the [ALA's Code of Ethics](https://www.ala.org/tools/ethics) states: 

> "We protect each library user's right to privacy and confidentiality with respect to information sought or received and resources consulted, borrowed, acquired or transmitted." 

While I deeply respect and admire this stance, it doesn't provide a solution for those of us wrestling with the aforementioned existential questions.

## A Data Problem

So, what kinds of data are we talking about here? The data I’m mostly
referring to here is the **circulation transactions** that are
recorded for patron activities such as checkouts, check-ins, holds
placed, holds filled, etc. The data contained within these types of
records are valuable because they represent a wide range of things:
* Where patrons' interests lie in terms of subsets of titles and items
* Which library locations are popular for what genres and item types
  (e.g. titles for children, titles for adults, etc.) 
* Which types of patrons and how many are utilizing our resources
* How many copies of a new title we should order based on past usage
  metrics  
* How we should re-balance number of existing copies among locations
* ...and so on — the list goes on

![Library Lending Cards](/lending-cards.jpeg "Olden Times" ){: .rounded-10 width="500" height="413" .w-50 .right}

“Just anonymize the data!” you may be saying to yourself. Sure, this works to a degree, but then you lose the number of unique patrons, or how many patrons are borrowing certain genres, or how much of the overall patron population uses a subset of resources — again, the list goes on and on. We want to know "**the who**" but not "**who the who is specifically**", nor do we want to know or remember for all the reasons stated earlier.

We need to strike a balance between 100% privacy protection of "having no data" and the statistical utility of "having data".

![Description:The chart visualizes a curved line that demonstrates the inverse relationship between Privacy and Analytical Utility. The x-axis represents Analytical Utility, and the y-axis represents Privacy. Both axes range from 0 to Origin Point: At the bottom-right corner of the chart, where the Analytical Utility is at its maximum (1.0), the Privacy is at its minimum (almost 0). An annotation labeled "Original Data" is located near this point, signifying that when the data is in its original, unaltered state, it has maximum analytical utility but minimal privacy. End Point: At the top-left corner of the chart, where the Analytical Utility is at its minimum (almost 0), the Privacy is at its maximum (1.0). An annotation labeled "No Data" is located near this point, indicating that when no data is available or shared, privacy is maximized, but analytical utility is nullified. Mid Point: Approximately halfway along the curve, there's an annotation labeled "Compromise." This point symbolizes the balance or trade-off between preserving privacy and maintaining analytical utility. ](/privacy_vs_utility_chart.svg "the inverse relationship between Privacy and Analytical Utility"){: width="720" height="432" .w-100}

## Pseudonymization

The [Wikipedia article on Pseudonymization](https://en.wikipedia.org/wiki/Pseudonymization) states that:

> Pseudonymization is a data management and de-identification procedure by which personally identifiable information fields within a data record are replaced by one or more artificial identifiers, or pseudonyms. A single pseudonym for each replaced field or collection of replaced fields makes the data record less identifiable while remaining suitable for data analysis and data processing.

Given this technique, we can now **strike a balance** in our quest to **obscure the personally identifiable information (PII)** of the patron while retaining analytical value by assigning a pseudonym.

Something like the following could be used to link a pseudonym to a patron:

| Patron ID         | Record Create Date | Patron Name  | pseudonyms |
|-------------------|--------------------|--------------|------------|
| 1 | 2017-05-21 | Chimperson, Chimpy H | patron1 |
| 2 | 2017-05-21 | Chimperson, Chimpy Jr | patron2 |
| ... | ... | ... | ... |
| 90042 | 2019-02-10 | Chimperson, Chimpette | patron90042 |

This may be good enough for most. But keep in mind the implications of doing it this way. In this example of pseudonymization, we're now required to maintain this extra data in the record itself, or in another lookup table so that any future data being processed produces the same consistent results. Also, in this particular implementation pseudonyms are being assigned to patrons as they're being created, revealing or "leaking" extra information about the patron that we may not feel comfortable with.

Of course these pseudonyms must be protected, as there is now a direct 1-to-1 relationship between the data subject and the pseudonym—in other words, patron activity can also be revealed by simply reversing the process. This data must also be maintained properly so that the integrity of the already processed data and any future data processed remains intact.

"**Is there a better way?!**" Of course there is!

## Cryptographic Hashing

Another way to approach this task of pseudonymization is by use of a cryptographic hash function — a one-way function that produces a "digest" of the input. This digest, or hash value, can not be reversed (at least not feasibly) from the output alone. These hash values are fixed-size alphanumeric strings that **uniquely identify the input**.



The ideal hash function has three main properties:

1. ***Easy to calculate*** **a hash for any given input**
2. ***Extremely difficult*** **or impossible to reverse, or "unscramble" the hash**
3. ***Extremely unlikely*** **that any two given inputs would produce the same hash value—they are unique.**

Property number three states that when two different inputs produce the same exact output — also known as a collision — this is to be considered *bad*. And if collisions happen with some degree of regularity, then that's considered to be *very bad* and you should look for another cryptographic hashing function immediately. **But, what if we could engineer the hash values to collide with some degree of higher probability ... on purpose?**

> Introducing a degree of uncertainty could be very good for our application of producing patron pseudonyms with the goal of increasing the protection of patron privacy. **If the probability is high — but not *too high* — that any two data subjects will share the same pseudonym, we will introduce a degree of uncertainty about any one data subject in our data**. Picking the right degree of probability for these types of pseudonym collisions can **preserve statistical value of the data while increasing the amount of privacy for library patrons**.
{: .prompt-tip}

How do we increase the probability of collision while having our data retain statistical value? It's actually fairly simple — we select a smaller number of bits from the overall output of the cryptographic hash.

First, it's an important realization to make that a good cryptographic hash function (like sha256) always results in the same output for a given input. For example:

```python
import base64
from hashlib import sha256
import requests

def my_hash(string_value):
  """
  returns a base64 encoded string value of the sha256 hash of an 
  input string
  """
  return base64.b64encode(
      sha256(
          string_value.encode('utf-8')  # bytes in UTF-8 encoding
      ).digest()
  ).decode('utf-8')

# sha256 hash of our string
my_hash('l33t-haxz0r')  # 'vAo3JcC6kgajn5lLDn5zRTCCeyJNoQGtSnf0LkD3wvU='

# same output as before
my_hash('l33t-haxz0r')  # 'vAo3JcC6kgajn5lLDn5zRTCCeyJNoQGtSnf0LkD3wvU='

# changing the string results in a totally different hash
my_hash('l33t-haxor')   # 'IGAQ9LRSA4I+yx9B1wH3/pal9TBQZaIqSX2G6ZddqHk='

# just to demonstrate that it's a fixed lenght value for any sized input
response = requests.get('https://www.gutenberg.org/ebooks/11.txt.utf-8')
my_hash(response.text), len(response.text)  
#  ('fTG/gW7DQvZ0g1Xu4N8iRXNpi9bWh6mL7oYOCuPgDOg=', 167711)
```

The other important thing to know about cryptographic hashing functions is that the resulting parts, or (literal) bits of the resulting hash value are totally independent of, and not influenced by any other bits in that same hash. The output **hash values are uniformly distributed over the output space**.

It means that:

> Every bit of a cryptographic hash is created equally, and **we can select as many, or as few a number of bits of the resulting hash value as we like! We can therefor control the number of "bins" that can be represented with the selected portion of the hash bits, and be virtually guaranteed even distribution across all of these bins**
{: .prompt-tip}

## The Birthday Paradox

So, ["Why is a raven like a writing desk?"](https://en.wikipedia.org/wiki/Hatter_(Alice%27s_Adventures_in_Wonderland)#Riddle) I haven't the slightest idea, but I do know we can apply this interesting paradox to our real-world application! 

The birthday paradox refers to the counterintuitive fact that, with only 23 people, the probability of at least two of them sharing the same birthday exceeds 50% (assuming birthdays are uniformly distributed among the population.) The [Wikipedia article on the "birthday problem"](https://en.wikipedia.org/wiki/Birthday_problem) is worth reading and goes into more detail on this interesting fact. As a bit of an aside, there end up being 253 *pairs* of people among just 23, which is higher than what would intuitively be assumed.

```python
# using a loop to count the pairs for 23 people

count = 0
for i in range(23):
    for j in range(i+1, 23):
        count += 1

print(count)  # 253
```

So, "where am I going with this?", and "HOW *is* a raven like a *writing desk*?!" you may be asking dear reader. Hang tight while we break this down a little bit further — and at least answer that first question.

![https://en.wikipedia.org/wiki/Birthday_problem#/media/File:Birthday_Paradox.svg](/the_birthday_paradox.svg "image from https://en.wikipedia.org/wiki/Birthday_problem#/media/File:Birthday_Paradox.svg"){: .right .w-50}

<div>If you consider each birthday to be a "bucket" and that we ignore leap years, then how many people are required before you have a \(99.999\%\) <i>probability</i> that <b>any two</b> persons will share the same bucket, or birthday — <b>in other words, what sample size of the population is needed for there to be a VERY high likelyhood of collision?</b></div>

There's a handy online calculator here at [kevingal.com/apps/collision.html](https://kevingal.com/apps/collision.html) to solve for some of these types of questions. I won't go too much into the math here, but it's interesting to play around with the numbers.

![https://kevingal.com/apps/collision.html](/hash_collision_calculator1.png "https://kevingal.com/apps/collision.html"){: .right. w-50}
So, as it turns out, the answer to the above question is a relatively low number — **only 90 people** are needed to be selected from the population before there is a very high degree of probability that they will land in the same bucket representing a birthday.

To apply some other example numbers to this, and get a little bit closer to understanding how we can take this paradox and make use of it for our case — lets say we wanted to find out how many buckets we would need for the following:

* <div>A patron population (\(k\), or items in the calculator) of \(5,000\) individuals</div>
* <div>A desired probability of \(99.999\%\) chance that any two patrons \(P\) will map to the same bucket, or collide</div>

<div>Using that same calculator and playing around with the numbers, we find that 20 bits, or \(2^{20}\)different "buckets" — \(1,048,576\) different combinations — would be needed to meet our requirements. The neat thing about this number of bits is that it neatly encodes into a 4-character printable token</div>

```python
import base64
import random

random_20_bits = random.getrandbits(20)

# convert the 20-bit integer to bytes (3 bytes for 20 bits in this case)
bytes_representation = random_20_bits.to_bytes(3, 'big')

# base64 encode the byte representation
base64_encoded = base64.b64encode(bytes_representation).decode('utf-8')

base64_encoded  # C6QF
```

Our patron population is however, a *tad* larger than 5,000 — we seem
to typically hover around 300,000 active patrons give or take.

Given this, what bucket size should we be looking for? It mostly
depends on two things:

1. What is our population size?
1. How many collisions we care to tolerate in our data.

Remember that we're only talking about the probability of **any two**
hashes colliding *approaching 100%*. Doing so with the fewest amount
of bits is also desirable since we'd also like to keep our token, or
pseudonym size low. This process should introduce deliberate
uncertainty, so just *how much* uncertainty to introduce is a bit
subjective.

<div>For my application, I'm picking 32 bits, which would provide \(2^{32}\) buckets or \(4,294,967,296\) values for the rough population size of \(300,000\) to evenly distribute across after hashing.</div>

That’s a lotta ~~cheddar~~ buckets!!!

There's also an approximation that can be made to see how many
collisions should be expected. The [Wikipedia page on the Birthday
Problem](https://en.wikipedia.org/wiki/Birthday_problem#Approximations)
provides more details.

<div>The expected number of collisions for \(n\) patrons in a hash space of size \(M\) can be approximated as:

\[
C \approx \frac{n^2}{2M}
\]
</div>

```python
# Given values
n = 300000  # Number of items
M = 2**32   # Number of possible hash values for 32 bits

# Calculate the expected number of collisions using the approximation
C_approximated = n**2 / (2 * M)

C_approximated  # 10.477378964424133
```

The result of the approximation calculation is under 11 collisions — in a population size of 300,000 this helps us retain statistical utility, while increasing patron privacy compared to other pseudonymization techniques.

## Stochastic Pseudonymization

I've termed this method 'Stochastic Pseudonymization' because it
generates pseudonyms with an inherent level of unpredictability. By
truncating bits from the SHA256 hash, a small portion of records might
produce the same hash value, or 'collide'. This intentionally designed
uncertainty enhances security by making it more challenging to
associate pseudonyms with their original records. I believe it strikes
a nice balance between patron privacy and statistical utility.

***Stochastic*** refers to the property of being well-described by a
random probability distribution. [https://en.wikipedia.org/wiki/Stochastic](https://en.wikipedia.org/wiki/Stochastic)

***Pseudonymization*** as we described earlier, refers to the
technique where personally identifiable information fields within a
data record are replaced by one or more artificial identifiers, or
pseudonyms. [https://en.wikipedia.org/wiki/Pseudonymization](https://en.wikipedia.org/wiki/Pseudonymization)

In the below Python implementation, I'm using the widely-used
[cryptography library](https://github.com/pyca/cryptography), which
includes the function I'm using to produce the hash: PBKDF2HMAC
(Password-Based Key Derivation Function 2 with Hash-based Message
Authentication Code). While the result of PBKDF2HMAC is more
accurately referred to as a "derived key", it does end up being
beneficial to use it for hashing here for a few reasons:
* A salt is created by combining parts of the patron record (exact
  create date, and the patron id) along with the app_secret, allowing
  us to generate the same consistent hash for a data subject.
* The SHA256 algorithm is applied multiple times to make reversing the
  hash even more computationally infeasible
* A specific output length (in my implementation, 4 bytes for our 32
  bit hash space) can be specified, simplifying the truncation of the
  hash bits.

### Implementation

```python
import base64
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import constant_time
import math
import os


class StochasticPseudonymizer:
  def __init__(
      self,
      app_secret,                 # protect this secret
      population_size=300_000,    # number of items
      target_probability=0.99999, # target collision probability
      iterations=100_000          # PBKDF2 iterations
  ):

    self.app_secret = app_secret
    self.iterations = iterations

    # Calculate the number of bins for the desired collision 
    # probability
    self.num_bins = int(
        self.calculate_num_bins(
            population_size,
            target_probability
        )
    )

    # Calculate the number of bits required
    self.num_bits = math.ceil(math.log2(self.num_bins))
    # Calculate the number of bytes required for the given 
    # number of bits
    self.num_bytes = (self.num_bits + 7) // 8

  @staticmethod
  def calculate_num_bins(population_size, target_probability):
    return population_size**2 / \
        (-2 * math.log(1 - target_probability))

  def generate_token(self, pii, patron_record):
    # Calculate the salt using PII, app_secret, and patron record 
    # fields
    salt = (
        str(pii)
        + self.app_secret
        + str(patron_record['id'])
        + str(patron_record['createdDate'])
    ).encode('utf-8')

    # PBKDF2 hashing with length determined by self.num_bytes
    hash_value = PBKDF2HMAC(
        algorithm=hashes.SHA256(),
        length=self.num_bytes,
        salt=salt,
        iterations=self.iterations,
        backend=default_backend()
    ).derive(
        str(pii).encode('utf-8')
    )

    # Convert the hash value to an integer and take it modulus
    # num_bins
    token_value = int.from_bytes(
        hash_value,
        byteorder='big'
    ) % self.num_bins

    # Convert the integer token value to bytes and then to a base64 
    # string
    return base64.b64encode(
        token_value.to_bytes(
            (token_value.bit_length() + 7) // 8, byteorder='big'
        )
    ).decode('utf-8').rstrip('=')
```

### Example Use

I'll take the Chimperson troupe from the example previously:

| Patron ID         | Record Create Date | Patron Name  | pseudonyms |
|-------------------|--------------------|--------------|------------|
| 1 | 2017-05-21 | Chimperson, Chimpy H | patron1 |
| 2 | 2017-05-21 | Chimperson, Chimpy Jr | patron2 |
| ... | ... | ... | ... |
| 90042 | 2019-02-10 | Chimperson, Chimpette | patron90042 |



```python
example_secret = """\
  monkey123 (please protect this secret ... and don't make it monkey123 !)
""".strip()

# instantiate the class
pseudonymizer = StochasticPseudonymizer(app_secret=example_secret)


patron_records = [
    {
        'id':1, 
        'createdDate': '2017-05-21', 
        'patronName': 'Chimperson, Chimpy H'
    },
    {
        'id':2, 
        'createdDate': '2017-05-21', 
        'patronName': 'Chimperson, Chimpy Jr'
    },
    {
        'id':90042, 
        'createdDate': '2019-02-10', 
        'patronName': 'Chimperson, Chimpette'
    }    
]

for patron_record in patron_records:
  token = pseudonymizer.generate_token(
      patron_record['patronName'], 
      patron_record
  )
  print(token)
```

And here are the tokens or pseudonyms produced!

```bash
BFgC9Q
31fGmw
MOyHUA
```

[GitHub Gist](https://gist.github.com/rayvoelker/56ff325286fd0b34870e82103a26bdb8)

## Thanks

Thank you for reading this post! It was a lot of fun to think more
deeply about this topic, and then write it all into a post. I hope
others find it useful and can apply the process to fit their own
needs. Drop me a line if you're among those that found this useful, or
want to offer any comments or corrections!
[https://twitter.com/ray_voelker](https://twitter.com/ray_voelker)

And of course, many, many thanks to Steve Gibson for expertly
explaining this on the Security Now! Podcast # 940

Below you can find links to the show, show notes, and transcripts.

* [twit.tv/shows/security-now/episodes/940?autostart=false](https://twit.tv/shows/security-now/episodes/940?autostart=false)
* [www.grc.com/sn/SN-940-Notes.pdf](https://www.grc.com/sn/SN-940-Notes.pdf)
* [twit.tv/posts/transcripts/security-now-940-transcript](https://twit.tv/posts/transcripts/security-now-940-transcript)

![how is a raven like a writing desk](/how_is_a_raven_like_a_writing_desk.webp){: .w-100}