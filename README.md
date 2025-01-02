# nosql-challenge

# UK Food Standards Agency Data Analysis

## Introduction

The UK Food Standards Agency evaluates various establishments across the United Kingdom and assigns them food hygiene ratings. As part of the "Eat Safe, Love" magazine, we've been tasked with analyzing this data to help journalists and food critics make decisions about where to focus future articles.

---

## Part 1: Database and Jupyter Notebook Set Up

In this part, you'll set up the database and confirm that the data has been imported correctly into MongoDB.

### Instructions:
1. Use the `NoSQL_setup_starter.ipynb` notebook for this section of the challenge.
2. **Import the Data**: Import the provided `establishments.json` file into your MongoDB instance. Name the database `uk_food` and the collection `establishments`.

   Example Terminal Command:
   ```bash
   mongoimport --db uk_food --collection establishments --file establishments.json --jsonArray
   ```

3. **Import Required Libraries**: In your Jupyter Notebook, import the following libraries:
   ```python
   from pymongo import MongoClient
   from pprint import pprint
   ```

4. **Create MongoClient Instance**:
   In your notebook, create an instance of MongoClient to connect to MongoDB:
   ```python
   mongo = MongoClient(port=27017)
   ```

5. **Confirm Data Import**:
   - **List the Databases**: To ensure the `uk_food` database was created, list all the databases in your MongoDB instance:
     ```python
     print(mongo.list_database_names())
     ```
   - **List Collections**: List the collections within the `uk_food` database to confirm the `establishments` collection exists:
     ```python
     db = mongo['uk_food']
     print(db.list_collection_names())
     ```
   - **Display One Document**: Use `find_one` to display a sample document from the `establishments` collection:
     ```python
     establishments = db['establishments']
     pprint(establishments.find_one())
     ```

---

## Part 2: Update the Database

In this section, you'll make several updates to the database, including adding a new restaurant, updating existing records, and performing data conversions.

### Instructions:
1. **Add New Restaurant**: An exciting new halal restaurant, "Penang Flavours", has just opened in Greenwich. Add the following details to the database:

   ```json
   {
       "BusinessName": "Penang Flavours",
       "BusinessType": "Restaurant/Cafe/Canteen",
       "BusinessTypeID": "",
       "AddressLine1": "Penang Flavours",
       "AddressLine2": "146A Plumstead Rd",
       "AddressLine3": "London",
       "AddressLine4": "",
       "PostCode": "SE18 7DY",
       "Phone": "",
       "LocalAuthorityCode": "511",
       "LocalAuthorityName": "Greenwich",
       "LocalAuthorityWebSite": "http://www.royalgreenwich.gov.uk",
       "LocalAuthorityEmailAddress": "health@royalgreenwich.gov.uk",
       "scores": {
           "Hygiene": "",
           "Structural": "",
           "ConfidenceInManagement": ""
       },
       "SchemeType": "FHRS",
       "geocode": {
           "longitude": "0.08384000",
           "latitude": "51.49014200"
       },
       "RightToReply": "",
       "Distance": 4623.9723280747176,
       "NewRatingPending": true
   }
   ```

2. **Find the `BusinessTypeID`**: Query the database to find the `BusinessTypeID` for `"Restaurant/Cafe/Canteen"` and display the `BusinessTypeID` and `BusinessType` fields.

   ```python
   result = establishments.find_one({"BusinessType": "Restaurant/Cafe/Canteen"}, {"BusinessTypeID": 1, "BusinessType": 1})
   pprint(result)
   ```

3. **Update New Restaurant with `BusinessTypeID`**: Update the `"Penang Flavours"` restaurant with the found `BusinessTypeID`:
   ```python
   establishments.update_one(
       {"BusinessName": "Penang Flavours"},
       {"$set": {"BusinessTypeID": business_type_id}}
   )
   ```

4. **Remove Establishments in Dover**: 
   - Count how many documents in the database have "Dover" as the `LocalAuthorityName`:
     ```python
     count = establishments.count_documents({"LocalAuthorityName": "Dover"})
     print(f"Number of establishments in Dover: {count}")
     ```
   - Delete all documents where `LocalAuthorityName` is "Dover":
     ```python
     establishments.delete_many({"LocalAuthorityName": "Dover"})
     ```
   - Verify the number of documents after deletion:
     ```python
     count_after = establishments.count_documents({"LocalAuthorityName": "Dover"})
     print(f"Number of establishments in Dover after deletion: {count_after}")
     ```

5. **Data Type Conversion**:
   - Convert the `latitude` and `longitude` fields from strings to decimal numbers using `update_many`:
     ```python
     from decimal import Decimal
     from bson import Decimal128
     establishments.update_many(
         {"geocode.longitude": {"$type": "string"}},
         [{"$set": {"geocode.longitude": Decimal128(Decimal("$geocode.longitude"))}}]
     )
     establishments.update_many(
         {"geocode.latitude": {"$type": "string"}},
         [{"$set": {"geocode.latitude": Decimal128(Decimal("$geocode.latitude"))}}]
     )
     ```

   - Convert `RatingValue` from strings to integers:
     ```python
     establishments.update_many(
         {"RatingValue": {"$in": ["1", "2", "3", "4", "5"]}},
         [{"$set": {"RatingValue": {"$toInt": "$RatingValue"}}}]
     )
     ```

---

## Part 3: Exploratory Analysis

In this part, you'll explore the database and answer key questions that the magazine editors need for their analysis.

### Instructions:
1. **Hygiene Score of 20**:
   - Find establishments with a hygiene score of 20 and display the first document in the results:
     ```python
     hygiene_20 = establishments.find({"scores.Hygiene": 20})
     print(f"Number of establishments with hygiene score 20: {hygiene_20.count_documents({})}")
     pprint(hygiene_20[0])
     ```

2. **Establishments in London with RatingValue >= 4**:
   - Use `$regex` to search for "London" in the `LocalAuthorityName` field and filter establishments with a `RatingValue` of 4 or higher:
     ```python
     london_establishments = establishments.find({
         "LocalAuthorityName": {"$regex": "London"},
         "RatingValue": {"$gte": 4}
     })
     pprint(london_establishments[0])
     ```

3. **Top 5 Establishments with RatingValue of 5**:
   - Find the top 5 establishments with a rating of 5, sorted by the lowest hygiene score, and located near "Penang Flavours" (within 0.01 degrees of latitude and longitude):
     ```python
     top_5 = establishments.find({
         "RatingValue": 5,
         "geocode.longitude": {"$gt": 0.07384000, "$lt": 0.09384000},
         "geocode.latitude": {"$gt": 51.48014200, "$lt": 51.50014200}
     }).sort("scores.Hygiene", 1).limit(5)
     pprint(top_5[0])
     ```

4. **Count Establishments with Hygiene Score of 0**:
   - Use the aggregation pipeline to count establishments by `LocalAuthorityName` with a hygiene score of 0:
     ```python
     aggregation_result = establishments.aggregate([
         {"$match": {"scores.Hygiene": 0}},
         {"$group": {"_id": "$LocalAuthorityName", "count": {"$sum": 1}}},
         {"$sort": {"count": -1}},
         {"$limit": 10}
     ])
     for result in aggregation_result:
         pprint(result)
     ```

---

## Conclusion

By following these steps, you'll have completed the analysis and provided valuable insights into the data for the editors of Eat Safe, Love. This analysis helps the magazine editors identify locations with great food hygiene, new potential food spots, and areas to avoid when focusing on their next food critique article.
```
