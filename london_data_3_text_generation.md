# Generating Property Listings with Machine Learning

<time datetime="2021-04-10 07:00">10th Apr 2021</time>

After scraping a dataset of rental listings, one of my first thoughts was to use this to train a neural network to generate property descriptions. I didn't think this automation would be useful for anyone, not even the laziest of estate agents, but it felt like an achievable task. That was the plan in 2018 and, a mere four years later, I've finally gotten around to doing it.

A more formal description of the task we're attempting is _conditional generation_. That is, getting a machine learning system to generate text _given_ certain inputs. Having a language model spit out arbitrary text is easier but far less useful. Conditional generation could be used for automating the production of quite prescribed quantitative documents.

The recent flurry of deep learning research for NLP has generated impressive gains. These have been driven by advances in network architectures and large (and I mean large) training datasets. While the resources required to train these state-of-the-art models from scratch is beyond the reach of mere mortals, the availability of open sourced models, and the development of high level APIs to fine-tune them, has meant that even randos writing blogs can join in on the fun. So here we are, but one man with a MacBook Pro, an AWS account, and a dataset of 27,000 rental listings. How far can we get with that?

## Preparing the model and data!

We will use the numerical and categorical data for properties (e.g. number of bedrooms, rent, amenities) to train a model. The goal is for the model to generate text that resembles the original property descriptions as best as possible. I say best as possible, I actually mean with as minimal inconvenience to me. So we will use the [transformers library](https://huggingface.co/transformers/), [Trainer API](https://huggingface.co/transformers/main_classes/trainer.html), and [model hub](https://huggingface.co/models) from Huggingface. If you haven't come across these but are reading this blog I assume you are a supportive family member. Hello.

We will fine-tune the [T5-large model](https://huggingface.co/transformers/model_doc/t5.html) released by Google. One benefit of this model is that it treats all tasks as a text-to-text problem, where we can define the input text as the property data written as a string, and the output as the text describing the property. Because the input can only be text, I decided to "hashtag-ify" the labels to give some sort of pattern for the property fields and values. I suspect the model would have trained fine without this, but hashtags are a thing now.

An example input would looks like...

`translate Hashtags to English: #West_Hampstead #rent_704 #professional #4_bedrooms #postcode_NW2 #flat #living_room`

And the model would learn to generate a description that looked like this... (note that this is the actual property listing, not the AI generated text)

> Double room available in a spacious lovely flat in a quiet residential area in West Hampstead. The flat is split in 2 floors, there are 4 bedrooms, lounge, kitchen and 3 bathrooms. You would be living in a bedroom upstairs with access to shared lounge, kitchen and bathroom. The room comes with a double bed and built-in closet. You would be sharing with 3 other professionals. We all have our own lives and schedules, but are also happy to have a chat or share a glass of wine if we happen to be home at the same time. We are looking for someone who is also a busy professional with similar lifestyle.

## Training the model

The model was trained using a stripped down script from Huggingface, on a p3.2xlarge EC2 instance on AWS, with all default training parameters (see "...with as minimal inconvenience to me..."). I was curious what we could achieve with such a straight forward implementation.

So how did the model go? Surprisingly well in fact. I'll cherry pick a few examples here, but feel free to peruse the 80+ test examples I run the model over (that is, examples that the model did not see during the training period).

_translate Hashtags to English: #Elephant_and_Castle #rent_645 #professional #4_bedrooms #postcode_SE1 #flat_

> Hello! We have a double room available in our flat in Elephant and Castle. The room is fully furnished with a double bed, wardrobe, chest of drawers, desk and chair. The flat is on the first floor of a modern building with lift access. There is a shared living room, kitchen and bathroom (with shower over the bath). The rent is £645 per month plus bills which are around £60-70 per month. We are looking for someone who is tidy, clean and respectful. We are looking for someone who is considerate and respects each other's privacy. Please get in touch

_translate Hashtags to English: #Wapping #rent_585 #balcony #bills_included #professional #garden #2_bedrooms #postcode_E1 #flat_

> I am looking for a flatmate to share my 2 bedroom flat in Wapping. The room is a good size double and comes with a double bed, wardrobe and chest of drawers. There is also a small balcony off the living room which is great for summer BBQ's. The flat is on the first floor of a modern block with lift access. It has a large kitchen/dining area with a dining table and chairs, and a bathroom with a bath and shower. There is a communal garden at the rear of the building, which is perfect for BBQ's in the summer

_translate Hashtags to English: #Whitechapel #rent_680 #bills_included #professional #3_bedrooms #postcode_E1 #flat_

> I have a double room to rent in Whitechapel. The room is fully furnished with a double bed, wardrobe and chest of drawers. The flat is located on the top floor of a Victorian house. It has a living room, kitchen and bathroom. There is also a separate toilet and shower room. The rent is £680 per month which includes all bills. You will be sharing the flat with two young professionals who are very friendly and easy to get along with. We are looking for someone who is clean and tidy and respectful. We are looking for someone who is considerate and respects each other

I don't imagine any of the above listings will make you jump out of your chair and slap down some money for a deposit, but it is pretty cool how much the model can customise the generated text. For example, the listings nearly always accurately state the number of rooms, as well as use that context to describe the current flatmates. Similarly, it often accurately states the rent in the text, as well as describing what the garden is like (if it is prompted). If you examine all the generated listings together you can see they are all quite formulaic. By increasing the "temperature" of the generated text, we can increase the randomness of the text, but then the output starts looking like maybe these listings were written by a drunk flatmate after a night out.

Because there is so much information baked into the T5 model, it can generalise to locations outside of the UK. Let's pick Manhattan...

_translate Hashtags to English: #Manhattan #rent_1200 #professional #2_bedrooms #flat #living_room_
This amazing apartment is on the 5th floor, so it is quite a high-rise.The room comes fully furnished with kingsize bed, double closet, table, and 4 chairs! The apartment has two bathrooms, one in each: one downstairs (with toilet), one upstairs (shower) and one of those are just for guests only!! The living area/bedroom includes huge windows looking over New York skyline from all directions...There are two bedrooms available; one large, spacious bedroom and one extra small but nice space to keep a dog.You will share the entire upper house

While it's obvious to us that Manhattan is in New York, it surprised me that this model was able to take this knowledge and combine it with the training we did to generate a listing. Pretty cool!

## What's next?

Nothing. I feel that 3 years is long enough to be pottering around with this project and consider this case closed. However, feel free to play around with the [fine-tuned t5-large model](https://t5-listing-generator.s3.eu-west-2.amazonaws.com/listing_generator.tgz) yourself to make sure I didn't just make these results up. I have chosen not to release the scraped data used here and the two previous posts. Sometimes users add their full names or phone numbers to the listings, and anonymising 27,000 documents was not something I liked the prospect of. Hope you enjoyed the read!
