# NLP analysis of London Rental Listings

<time datetime="2018-04-07 07:00">7th Apr 2018</time>

## Preamble

There is a real knack to finding an acceptable flat in London on a reasonable budget. The problem which I highlighted in the previous article was that there is so much potential for just terrible, terrible value. In my first round of London flat viewings I was perhaps a little naive. When looking at property listings I didn’t pay much attention to the descriptions given, aside from the basic information necessary, and focused more on the photos. The photos should give a better indication of what the flat is like… right? In that first round of viewings I had the unusual experience of going to shittier flats further and further out from the city, yet the rent was slightly increasing? It seems some unscrupulous estate agents felt they could charge whatever they wanted and, no biggie, some sucker would still take the flat in a week or two. They would be correct. I was that sucker.

I was determined to do a little better with my next round of flat hunting a year later, so I took full advantage of the filters and options available online to immediately remove any of the rubbish out there on the web/London. One of the most important filters I found was to only search through listings which were advertised by one of the flatmates of the property (as opposed to an estate agent or landlord). I wanted to find a place which had that sort of familial vibe, where the whole flat was hanging out and getting along. So obviously a flat with other people who wanted the same thing would probably take the time and effort to write their own listings, and really let their personality shine through in trying to attract someone likeminded.

Here, using the same data set I gathered for Part 1, we are going to look at the descriptions written for these listings. I want to run through some natural language processing (NLP) techniques to see if a computer can also pick up on the differences which I certainly have noticed. We can examine this by testing the ability of a computer to predict whether an estate agent, flatmate or landlord wrote a property listing based on the text alone. Specifically, what we’re doing is a supervised learning classification task. A few machine learning models will be trained on labelled data, that is the flat descriptions (the features), along with the associated author of those descriptions (i.e. agent, flatmate, landlord - the labels). Following training, we can use these models to try to correctly identify the author of descriptions they have not seen. Going into this, I was a bit unsure how well these models could perform because there should be a lot of similarities between all the descriptions. They will all talk about the rooms and how they’re furnished, the amenities, the neighbourhood. Will there be enough linguistic differences which the models can pick up? All the code I used for this analysis can be found over on my [GitHub](https://github.com/calamont/London-data-analysis).

## Bag of Words

So how does NLP actually work. There’s a contradiction here when we think about how well a computer should actually perform for such tasks. On the one hand, all language simply represents is information, which is a computer’s bread and butter. On the other hand, language represents information not so simply. There are inconsistencies, ill-defined definitions, and exceptions. And irony? Even a lot of humans can’t get their head around irony… Here we are making the task easier for ourselves and are merely trying to classify the documents, not infer any meaning from them. The most basic way to achieve this is with a “bag of words” (BOW) approach. We will rummage through the text, remove really common, non-informative words (called “stop words”) while also grouping very similar words together (porter-stemming). Then we tally up the word-count of all the different words in each example of text. It is also useful to normalise these values with respect to the word frequency within a particular block of text and within the wider corpus used for training. In doing this we can put more emphasis on more informative words over common words, such as house, flat, rent etc, used across the database. No spacial/temporal information is kept, its just… well its just a bag of words. It sounds a bit too simple to work, but for classification purposes it does just fine. The classic example is sentiment analysis on movie reviews. Obviously if someone uses the words “terrible”, “lame” and “super lame” a lot, chances are they didn’t like the movie.

I’m trying out three different models here. A naive Bayes, a support vector machine (SVM) and a random forest classifier. I don’t imagine anyone came here for a maths lesson, but I’ll give you a flavour of what each of these techniques are doing. As machine learning and AI permeates more into our everyday lives, I feel a good and curious member of humankind should have a level of familiarity with what’s going on under the hood.

**Naive Bayes:** Our feature set simply consists of a large vocabulary of words. The basic premise of the naive Bayes method for classification is that the presence of each word is considered independent of any another, which.. isn’t really how language works but.. shutupletsjustgowithitdontaskquestions! Naive Bayes uses conditional probability to work out what’s the most likely classification given the features that have been fed to it. Going back to the example of sentiment analysis in movies reviews, as the number of times somebody types “lame” into their IMDB review, the higher the probability the review is negative.

<img align="right" style="width: 50%; padding: 20px;" src="images/london_nlp_svm_max_sep_hyperplane_with_margin.png" />

**Support Vector Machine:** SVMs are not as simple to explain as naive Bayes, though they can be a lot more powerful. Abstractly, one can imagine if data from different classifications were plotted out, it may naturally appear grouped into separate clusters. The SVM tries to define a line (or “hyperplane”) which best draws a division between these different clusters. The support vectors are the closest data points from that hyperplane, which the SVM is trying to keep a maximal distance away from. If new data appears on one side of that line, its given some label, if it lands on the other side, it gets another label.

**Random Forest Classifier:** Let’s start with decision trees. Decision trees are that sort of flow-chart where you hit a node and you’re typically posed a yes/no question with two separate branches representing the possible answers. One of the branches is followed till you hit the next node with another question. You keep flowing through all these branches until you arrive at the end. The succession of questions should eventually lead you to the correct decision/classification/diagnosis. Naturally, the most important and informative classifying features are differentiated in the earlier nodes. For our task this can be thought of as posing questions about the words used in the text (i.e. is the word “luxury” used or not? No? Is the word “messy” used or not? etc. etc. etc. ….Ok you’re reading a description written by an estate agent!) Random forests are an ensemble method, which essentially builds many many of these decision trees using a subset of our samples and features. Each of these trees will have different words at higher or lower positions, and thus may lead to different classifications. These trees are averaged together to return a single classifier.

Using scikit-learn on Python makes the task of building a classifier so incredibly easy. There’s no need to reinvent the wheel here, so considering someone else has already taken care of implementing all the maths required we can just get a move on and start seeing some results. While scikit-learn also makes it very easy to optimise these models I’ve kept things pretty bare bones here for the sake of simplicity. So how does it all actually go? The accuracy of the models to identify the correct author of the descriptions is given below. The SVM and random forest both show pretty decent out-of-the box performance.

**Accuracy (3 classes / Agents v. Flatmates)** <br/>
**NB:** 75.5% / 91.8% <br/>
**SVM:** 84.1% / 95.6% <br/>
**RFC:** 83.5% / 95.2% <br/>

_Side note: Here I am simply showing the accuracy of each model, which is the number of predictions a model got correct divided by the total number of predictions. A more useful metric of performance is looking at recall and precision, which starts taking into account false positives and negatives. However, to explain this would then lead me into a whole meta-argument about weighing up these two opposing forces and how, as a population, we should understand these factors to better use statistical reasoning to make informed judgements for society and… ahh! See this is why I’m just using accuracy for now. In a future post I’ll get into the weeds._

Naturally, the more classes we are trying to predict, the more confused our models may become, which will degrade performances. Additionally, the differences that separate landlords and estate agents or flatmates may be subtle. Some landlords might advertise themselves more as a friendly flatmate, while others may take a more professional approach and be estate agenty. For this reason I've also included performances when just trying to split the difference between descriptions written by agents and flatmates. As can be seen in the above results, our models perform much better with this simplified task, with the SVM correctly identifying the author 96% of the time. But what exactly are the differences which are delineating these two classes. Well, we can go back to our naive Bayes classifier to help get an insight into this. Remember, this model looks at each word independently of the others and learns the probabilities that a description was written by an estate agent or flatmate, given the particular words used. As a result, we can flip this on its head and find out what words in the learnt vocabulary are most informative in classifying the description’s author. Out of the top 50 words for each class, I’ve selected what I think are the most interesting 25.

_Agent:_ relocation, reservation, delay, booking, estates, my deposit, waster, lettings, delighted, disappointment, reserve, pleased, ultimate, providers, client, present, displease, benefitting, magnificent, lowest, deserve, proud, particulars, deducted, apportioned

_Flatmate:_ hang, outgoing, yoga, sad, drink, laugh, Australian, film, respects, replace, chilling, Irish, occasionally, obviously, chat, chilled, bbqs, music, likes, relatively, co-op, chatting, happily, bunch, dinners

The difference here is pretty clear. Agents appear a bit more about business and swing from using super boring words to super positive sentiments. One can even put together a banal description based off these words. “We are proud/pleased/delighted to present this magnificent flat. Make a reservation now to discuss the particulars and get this location and avoid disappointment!” Whereas the flatmates clearly seem super chill! Who wouldn’t want to live with these fun guys? The one odd word out here is “sad”, but we’ll review that one later on. Not surprisingly we see Australia make a pretty high appearance on this list \*sigh\*.

## Word Embeddings

The bag of words approach is pretty dumb, and expensive. Also the way we store the information, as a position in a giant matrix, doesn’t convey any of the information contained within the word. A neat trick to get around these issues is word embedding, which allows the words to be stored as a point in space. A way to do this is to train a neural network to learn the words commonly associated with each other, which is possible as similar words (positive, negative, verb, adjective etc.) often get placed in the same contexts in sentences. Given enough sentences, the neural network can pick up on these similarities between words. Then one can extract some of the linear algebra used by these models to give a representation of the word in n-dimensional space. In doing this, the words are now vectors and we can start doing maths with them, such as finding which words are closest, or identifying clusters of words commonly used in one form of text or from one author. I found [this article by Chris McCormick](http://mccormickml.com/2016/04/19/word2vec-tutorial-the-skip-gram-model/) to give the clearest description of this process. To give you a flavour of word embedding in action, I wrote up a model which spits out its progress as it is trained over 1 million training steps. It isn’t perfect but we can still see what words the model thinks are closest to “three”.

```
Initialized
Nearest to "three" after 10 steps: repost therefor rotation transporting dwell critic teak iron
Nearest to "three" after 1000 steps: iron reference send call washing privacy repost oven
Nearest to "three" after 10000 steps: reference depot cut oven effective diner goods fully
Nearest to "three" after 100000 steps: two four one gritty tab second cocktail single
Nearest to "three" after 200000 steps: two four five one fitting couple all single
Nearest to "three" after 300000 steps: two four five fitting one x bathroom couple
Nearest to "three" after 500000 steps: two four five fitting one six x few
Nearest to "three" after 1000000 steps: two four five six one fitting x russia
```

Naturally, other numbers get used in similar contexts and, as a result, they appear closer when plotted in space. And “Russia”? Well… I don’t know. Maybe Russians live in threes around London. What if we look at the words closest to “student”?

```
Nearest to "student"
[('mature', 0.7892049551010132),
 ('worker', 0.7318430542945862),
 ('graduate', 0.7258175015449524),
 ('students', 0.6891041994094849),
 ('female', 0.6884569525718689),
 ('woman', 0.6866834759712219),
 ('postgraduate', 0.6825687289237976),
 ('sharers', 0.6713240742683411),
 ('male', 0.6585562825202942),
 ('studying', 0.6475297212600708)]
```

The number spit out is the [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity) used to determine which vectors (words) are closest/most similar. I think this is really cool! It’s pretty amazing that a computer can now “learn“ meanings behind our arbitrary human symbols without anybody having to hard-code anything into the machine. Now let’s go back and see why the hell the word “sad” was so informative for the naive Bayes classifier...

```
Nearest to "sad"
[('gutted', 0.6801810264587402),
 ('goodbye', 0.6487263441085815),
 ('leaving', 0.6377925276756287),
 ('tends', 0.6365676522254944),
 ('will', 0.598455011844635),
 ('must', 0.5809981226921082),
 ('decided', 0.5763540863990784),
 ('ll', 0.5608475804328918),
 ('enjoyed', 0.5578033924102783),
 ('fluffy', 0.5569889545440674)]
```

It turns out sad was being used in the context of a previous flatmate leaving i.e. “Sadly Bruce’s visa has expired and he must say goodbye and return to Australia“. I think perhaps we can ignore "fluffy".

One method of using word embeddings is to cluster similar words together, then use a BOWs style approach to see which clusters are more used by agents/flatmates and can help classify a given text. Another way is to average all the word vectors in a given property description together, then try to learn associations between the average vectors given by agents and those by flatmates. But instead, let’s take a step further with paragraph/document vectors. The general approach here is similar to what was achieved with word vectors, except now we try to instill the essence of a whole block of text into a n-dimensional vector, as opposed to individual words.

Doing this only achieved an accuracy of about 70%. It seems that this more fancy approach hasn’t performed any better than the BOWs method used at the very start. A reason for this may be that all the descriptions will have a similar structure, describing the flat, the location and any existing tenants. As such document vectors determined for agents, landlord or flatmates may be very samey. However, as we saw with the naive Bayes classifier, the specific words used by each of these authors may be incredibly different. Therefore, keeping this specific information, as is achieved with BOWs, allows us to make better classifications. I will note that researchers have generally found document vectors to be better than BOWs, which might be due to a difference in this data set or my limited experience with the technique.

## Comparing Text

Despite the poor performance here, document vectors still have some utility. Using these vectors, we can find the most "similar" example to a given block of text. Below, out of all 27,000+ desciptions in the corpus, the cosine similarity of Listing B was the closest to that of Listing A. Even if the classification accuracy of the document vectors wasn’t perfect, it certainly is picking up on similarities in these two descriptions.

**Listing A**

> _Double room in spacious 2 bed apartment. My current flatmate will be moving out at the end of December/beginning of January. I am therefore looking for a fun, friendly female professional who is clean, tidy and a non-smoker to take her room._

> _The apartment itself: On offer is a lovely double room, which comes fully furnished with plenty of storage space (fitted wardrobes and chest of drawers). The apartment is very well furnished including a fully fitted modern kitchen, with gas cooker, fridge, freezer, washing machine and all cooking utensils. The lounge/living room is bright and spacious, perfect for chilling in front of the TV or enjoying a relaxed dinner and a glass of wine. The bathroom has a bath/shower with a separate toilet_

> _Location: A 2-3 minute walk to Clapham South Station, close to all local amenities; shops, bars and restaurants. It is just off Clapham Common, which is great for runs and chilling in the summer and less than a 10 minute walk to Balham and Clapham's Northcote Road._

> _About me: I'm currently working as an Insurance Broker in the City. Outside of work I enjoy relaxed evenings at home, cooking, gym classes and taking advantage of London life._

> _Who I am looking for: A female professional who is clean and tidy. Would be happy to spend some evenings together, whilst respecting each others space and interests. Rent and Bills: 825 pcm plus bills, which come to around 75-100, incl. cheap Wandsworth council tax, electricity, gas, water, internet and TV license. If you are interested please drop me a message with the following details:- Name:- Age:- Occupation:- Interests:- Contact details:_

> _The current tenancy agreement will finish in 5 months time, but I am ideally looking for someone that would move in with a view to renewing in June 2018. I will get back to you at my earliest convenience to arrange a viewing. Look forward to hearing from you, and excited to find a new roommate :)_

**Listing B**

> _Looking for a lovely new flat mate to move in to a spacious flat at the end of January. Ideally looking for a like minded girl between the age of 28-35. I am a friendly, fun and sociable girl working as a primary school teacher in Notting Hill. I work hard during the week but enjoy socialising at the weekend. Sometimes with a glass of wine and a film but we also enjoy going out with friends. I am looking for someone who is clean and tidy but not obsessive. Also, I do spend a couple of nights a week at my boyfriends house._

> _I have a lovely house cat called Lola who is very friendly and provides good cuddles after a bad day, it is really important that you are comfortable around cats and not scared of them. The room is furnished with a double bed with storage space underneath, large chest of drawers and built in wardrobe. There is also an additional cupboard in the hall for extra storage space. The flat is situated within 5 minute walk to both Ealing Common and Acton Town tube stations. Ealing Broadway is about a 10 minute walk away. The flat has three double rooms, bathroom with shower and separate loo. There is a large living room with sky TV & dining area. The kitchen also has a back door which leads on the a small balcony with plants and table & chairs. We are also lucky to have access to large communal gardens which are beautifully kept and Ealing Common in a 2 minute walk._

> _Amenities: Broadband, Sky, landline telephone, washing machine, dishwasher, residents parking on the road with a permit.Communal gardens, bike shed and recycling on site. The rent for the room is £550 pcm MON-FRI including council tax, water, sky TV, internet and Netflix, Electricity and Gas. A deposit of one months rent will be needed to secure the room and in case of any damage during tenancy. If you think this sounds like the place for you please get in touch via email and include your contact number. Due to working hours I will not be including my number on the advert._

You might be thinking that "don't most property descriptions look like that?" Not necessarily, especially if we're considering all of the descriptions written by landlords and estate agents. I've included a random example below which shows how different these descriptions can be, and how impressive it is that the document embedding technique was able to identify the similarity between Listing A and Listing B.

**Listing C**

> _To live with 2 girls, 1 guy, to cover rent/bills while I'm away - easy going, friendly, clean. Amazing price for the area, 10 mins walk from clap sth or balham tube. Male preferred but not vital - come and meet us one evening next week! Returning to london around mid march, room available from late december possibly sooner!_

## Dishonest Agents

Using this document embedding technique we can also root out lazy lazy lazy estate agents who can’t be bothered writing a unique description for the flat they’re advertising (tsk tsk tsk). Funnily, I feel the below example is genuinely useful in better understanding how word embedding works, as words like "phenomenal", "astonishing" and "irresistible" are all used in a similar context (or in this case exactly the same context), which is what allows the model to understand they have a similar meaning.

> _OVERWHELMING Spacious Double room in PHENOMENAL apartment FEW MINUTES from MUDCHUTE station provides an EASY ACCESS to CANARY WHARF or GREENWICH and EXCELLENT connection to CITY CENTER. This TERRIFIC Flat comes with MODERN NEW furniture. Large MODERN FULLY FITTED kitchen with DINNING area bathroom and BALCONY. All bills included (council tax, water, electricity & gas). Free Internet Free cleaning service Tube and bus station few mins away Nice and friendly flatmates Working professionals and students are welcomed Relocation service \* ONLINE BOOKING AVAILABLE CALL WHATSAPP or TEXT me NOW! DONT DELAY! AVAILABLE for SHORT LET FOR Decmber-February-£165, from March-£200. Barbara_

> _IRRESISTIBLE DOUBLE room in New ASTONISHING flat few minutes to SOUTH QUAY or CANARY WHARF which provides GREAT connection to City Centre and EASY ACCESS to move around London. This FABULOUS flat is NEWLY refurbished with MODERN NEW furniture. LUXURY FULLY FITED kitchen bathroom and shared DINNING Area. All bills included (council tax, water, electricity & gas). Free Internet Free cleaning service bus, tube-few mins away Nice and friendly flatmates Working professionals and students are welcomed Relocation service CALL or TEXT me NOW! Barbara_

> _OVERWHELMING ENSUITE room in New EXTRAORDINARY MODERN apartment FEW MINUTES from CROSSHARBOUR station gives you an EASY ACCESS to CANARY WHARF or GREENWICH and GREAT connection to CITY CENTRE. The area is surrounded by bars, restaurants and supermarkets. This FABULOUS Flat comes with MODERN NEW furniture. Large LUXURY FULLY FITTED kitchen with DINNING area 2 bathroomS and 3 TOILETS GARDEN and PARKING. All bills included (council tax, water, electricity & gas). Free Internet Free cleaning service Tube and bus station few mins away Nice and friendly flatmates Working professionals and students are welcomed Relocation service \* ONLINE BOOKING AVAILABLE December to February promotion £180pw CALL WHATSAPP or TEXT me NOW! DONT DELAY! Barbara_

> _DAZZLING LARGE Double room in FABULOUS flat FEW MINUTES walk to SOUTH QUAY station or to CANARY WHARF which provides an easy access to move all around London and GREAT connection to CITY CENTER. This FABULOUS flat is NEWLY refurbished with MODERN NEW furniture. FULLY FITTED kitchen WITH DINNING area bathroom 2 toilets and LARGE GARDEN All bills included (council tax, water, electricity & gas). Free Internet Free cleaning service bus and tube station-few mins away Nice and friendly flatmates Working professionals and students are welcomed Relocation service CALL WHATSAPP or TEXT me NOW! DONT DELAY! AVAILABLE FOR SHORT LET FOR £165. Barbara_

> _ASTONISHING double in LUXURY house short walk to CROSSHARBOUR which provides an EXCELLENT connection to CITY CENTRE and an EASY QUICK ACCESS to BANK CANARY WHARF or GREENWICH and to move all around London. This OUTSTANDING house comes with MODERN furniture. LUXURY FULLY FITTED kitchen with DINNING area one bathroom and TERRACE. All bills included (council tax, water, electricity & gas). Free Internet Near station (bus, tube)-few mins away Nice and friendly flatmates Working professionals and students are welcomed Relocation service 2 lovely cats. CALL WHATSAPP or TEXT me NOW! Agency fee £150_

> _INEXPENSIVE Double rooms in FASHIONABLE flat next to WEST SILVERTOWN station which gives you an easy access to move around all London and GREAT connection to the City Center. The flat is newly refurbished with MODERN NEW furniture. modern kitchen and bathroom All bills included (council tax, water, electricity & gas). Free Internet Free cleaning service Near station (bus, tube)-few mins away Nice and friendly roommates Working professionals and students are welcomed Relocation service CALL, WHATSAAP or TEXT me NOW! AVAILABLE also for SHORT LET FOR £160/£170 Barbara_

> _ASTONISHING HUGE Double rooms in EXTRAORDINARY flat short walk to SOUTH QUAY station or to CANARY WHARF which provides an easy access to move around London and GREAT connection to CITY CENTER. The flat is NEWLY refurbished with MODERN NEW furniture. FULLY FITTED kitchen WITH DINNING area bathroom 2 toilets and GARDEN All bills included (council tax, water, electricity & gas). Free Internet Free cleaning service Near station (bus, tube)-few mins away Nice and friendly flatmates Working professionals and students are welcomed Relocation service AVAILABLE for short let until february for £165/from march £205 CALL WHATSAPP or TEXT me NOW! Barbara_

Perhaps be skeptical next time you read a listing for an astonishing room with easy access to the city centre. I see what you’re doing, Barbara… ಠ_ಠ

## Takeaways

NLP is a tool which will become omnipresent in our lives going into the future. Up until recently, the inherent “humanness” embodied in our language has thwarted the consistent logic of computation. With this impediment removed, the possibility of automation can include legal discovery and medical diagnostics from symptom descriptions. More indisously, as computers become more attuned to the meaning of our words, the clear separation of our digital technologies as tools may be jeopardised, with us being conditioned to think of them as companions, à la Her (2013). Hopefully by that stage I won't still be unreasonably preoccupied with thinking about estate agents and rent in London.
