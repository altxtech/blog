---
title: "How to develop API data extraction job with Go"
date: 2023-08-19:00:00-03:00
draft: true
---
## Intro

The task is very simple. You want to extract data from and API and load it into something like block storage,
or a database.  
On the surface level, it sounds like the type of thing that *should* be simple. And it is... for smaller extraction jobs
that take a few minutes at most to complete. However, when it comes to larger extraction jobs (hours to complete), and
making it a resilient, reliable, production-ready solution, it gets suprisingly complicated very fast.

### What is this guide  

This guide is meant to explore the core issuesa and solution of developing API data extraction jobs or ETL pipelines. 
These concepts are agnostic to programming languages and will apply to whatever programming language you chosse. However, this guide will use the Go 
programming language for code examples, and also explore Go features and tricks to deal with API extraction challenges.

### What makes a data extraction program production ready?

**At a minimum**, for a production environment, a data extraction should achieve at least these 3 things:
1. **Recover from failures.** For an hours-long job, it is *not viable* to restart it from scratch everytime there is an unforseen edge case exception or a random system outtage. Also, your program should be able to anticipate and handle some basic errors.
2. **Easy to debug.** On a large enough job, the law of large numbers clearly states that errors are *guaranteed* to happend. There is no going around that, but you should be able to know when and why the extraction failed, and possibly estimate an SLA for the extraction (is it 99% accurate? Or 99.9999%? You should be able to tell).
3. **Retry failed requests**. If you know which requests failed and why, you should then be able to fix your code and retry those, making your extraction as reliable as possible.  

**Idempotency is desirable, but not required**, which means that running a part of the extraction multiple times will
have the same end result as running it only once on wherever you're storing your data. In the data world, in practice,
not having idempotency will mean having duplicate data in your destination. You can handle deduplication downstream, but itis nice having it handled at the extraction step. Besides, making your replicated data a 1-for-1 to the source, opens some
new alternatives to auditing the result.  

**Do not worry about performance initially**. It doesn't matter if the extraction takes long as long as it is reliable.  
Your engineering hours are more valuables than some VM-hours. So, don't feel tempted to mess with asyncio if the
extraction is feasible without it. Would only be worth it if would save several days of runtime OR if you want to run 
the job multiple times. 

### Should you even build it?

Before you start coding, it is worth mentioning that maybe you could just... not. For most popular SaaS applications,
there are pre-built connectors offered by ETL/ELT platforms such as [Fivetran](https://www.fivetran.com/), 
[Stitch](https://www.stitchdata.com/), [Airbyte](https://airbyte.com/), among others.  

These platforms often offer generous free tiers, and even if you have to pay, in practice the cost of these platforms
will be less than the opportunitty cost of your engineering hours.  

If you are building an data migration or data connector for the API of a popular service and you were unfamiliar with
the products mentioned above, you should definitely take a look at those first. 

With that said...  

You can't always find a suitable connection for your business needs. Maybe because we're talking about a smaller,
unpopular, niche platform, or because the connectors offered by vendors do not meet some business required.  

In those cases, we are going to need to code it ourselves. So let's get to it.

## Case study  
I will use the Meta Markeing API as a case study. Let's imagine a client hired us to extract all their ads data and back it up to GCS.

## Starting point  

Let's assume you already have a script that handles doing the API calls and pagination. You may have a script that looks a bit like this:
**Obs:** I will be using the Meta Marketingasd

{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"log"
	"cloud.google.com/go/storage"
	"github.com/Valgard/godotenv"
)

func main() {

	// Load environment variables
	godotenv.Load(".env")

	// Create google cloud storage client
	ctx := context.Background()
	gcs, err := storage.NewClient(ctx)
	if err != nil {
		log.Fatal(err)
	}
	bucket := gcs.Bucket(os.Getenv("BUCKET_NAME"))

	// Build the initial request
	params := url.Values {
		"limit": { "100" },
		"date_preset" : { "maximum" },
		"access_token" : { os.Getenv("ACCESS_TOKEN") },
	}
	params.Add("field", "id")
	params.Add("field", "name")
	params.Add("field", "created_time")
	// ... And whatever other fields you need
	baseUrl := "https://graph.facebook.com/v17.0/" + os.Getenv("AD_ACCOUNT_ID") + "/ads"
	req, err := http.NewRequest("GET", baseUrl, nil)
	if err != nil {
		log.Fatal(err)
	}
	req.URL.RawQuery = params.Encode()

	const prefix string = "ads/"
	var key string


	// Request with pagination
	for page := 1; true; page++{

		// Execute the request
		fmt.Printf("Extracting page %d\n", page)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			log.Fatal(err)
		}
		defer resp.Body.Close()

		// Parse the json response body
		var json_data map[string]interface{}
		bytes_data, err := io.ReadAll(resp.Body)
		if err != nil {
			log.Fatal(err)
		}
		json.Unmarshal(bytes_data, &json_data)

		// Save the results for this page on GCS
		key = fmt.Sprintf("%d.json", page) // OBS: This key is PURPOSEFULLY bad. Will discuss why later...
		w := bucket.Object(prefix + key).NewWriter(ctx)
		if _, err := w.Write(bytes_data); err != nil {
			log.Println("Error saving page to gcs.")
		}
		if err := w.Close(); err != nil {
			log.Println("Error closing GCS Object writer")
		}

		// Check if there is a next page
		next := json_data["paging"].(map[string]interface{})["next"]
		if next == nil {
			// End extraction if not
			break
		}
		
		// Build next request
		req, err = http.NewRequest("GET", next.(string), nil)
		if err != nil {
			log.Fatal("Error building request")
		}
	}
	fmt.Println("Extraction ended.")
}
{{</ highlight>}}

## Rate Limiting

The code in the previous section will be enough for small data extraction jobs (few minutes to extract). However, the first issue you are probably going 
to run into is exceeding the API's rate limit. 
For most APIs this error will result ina [429 - Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429) status 
code (but not all APIs abide to this convention).  

To circumvent this issue, it's necessary to implement a *backoff-and-retry* loop. Some APIs send a `Retry-After` response headers to know how long you 
have to wait before resuming to send requests. If that's not the case, the recommended approach it to implement [Exponential Backoff](https://cloud.google.com/iot/docs/how-tos/exponential-backoff).  

We can achieve this by creating an `execute_request` function that encapsutales that logic.

{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
func main() {

// ...Previous code unchanged

// Request with pagination
	for page := 1; true; page++{

		// Execute the request
		fmt.Printf("Extracting page %d\n", page)
		resp, err := execute_request(req) // <--- Notice here!
		if err != nil {
			log.Fatal(err)
		}

// Remaining code unchanged...

}
{{</ highlight >}}

### With Exponential Backoff
{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
func execute_request(req *http.Request) (*http.Response, error) {

	backoffTime := 100
	maxRetries := 9
	var resp *http.Response

	for i := 0; i < maxRetries; i++ {

		// Try to execute the request
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return resp, err 
		}
		defer resp.Body.Close()

		// Check the status code
		switch resp.StatusCode {
			case 200:
				return resp, nil
			
			// ...Any cases that need special handling

			case 429:
				// Backoff
				fmt.Printf("Request failed with status code %d. Retrying in %dms\n", resp.StatusCode, backoffTime)
				time.Sleep(time.Duration(backoffTime) * time.Millisecond)
				backoffTime *= 2
				continue
		}
	}
	return resp, fmt.Errorf("Retry limit exceeded")
}
{{</ highlight >}}

### With the Retry-After header
{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
func execute_request_with_retry_after(req *http.Request) (*http.Response, error) {
    maxRetries := 9
    var resp *http.Response

    for i := 0; i < maxRetries; i++ {
        if resp != nil {
            resp.Body.Close() // Close previous response body before reusing the variable
        }

        // Try to execute the request
        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            return resp, err
        }
		defer resp.Body.Close()

        // Check the status code
        switch resp.StatusCode {
        case 200:
            return resp, nil
        // ...Handle other cases
        case 429:
            retryAfterHeader := resp.Header.Get("Retry-After")
            if retryAfterHeader != "" {
                retryAfterSeconds, err := strconv.Atoi(retryAfterHeader)
                if err == nil {
                    fmt.Printf("Request failed with status code %d. Retrying after %d seconds\n", resp.StatusCode, retryAfterSeconds)
                    time.Sleep(time.Duration(retryAfterSeconds) * time.Second)
                    continue // Retry after the specified time
                }
            }
        }
    }
    return resp, fmt.Errorf("Retry limit exceeded")
}
}
{{</ highlight >}}

### API specific solutions  

Like stated before, some API won't follow the convention of sending 429 HTTP Status Code upon rate limiting errors.  
So sometimes you need to implement the custom logic of whatever API you're working with. For instance, the Meta Marketing API only uses 400 status codes,
with their own custom error codes in the message body.  

In this case, since the response body can be read only once, we will have to load it's data and return it instead of the `http.Response` object.  

{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
func execute_request(req *http.Request) (map[string]interface{}, error) {

	// Create objet to write results into
	var data map[string]interface{}
	
	backoffTime := 100
	maxRetries := 9
	for i := 0; i < maxRetries; i++ {

		// Try to execute the request
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return data, err
		}
		defer resp.Body.Close()

		// Parse the json data and return
		content, err := io.ReadAll(resp.Body)
		if err != nil {
			return data, err
		}
		json.Unmarshal(content, &data)

		// Return if the request was successfull
		if resp.StatusCode == 200 {
			return data, nil
		}
		
		// Evaluate and handle the error
		switch data["error"].(map[string]interface{})["code"].(float64) {
			// Rate limit error
			case 17:
				fmt.Printf("Rate limit exceeded. Backing off by %dms\n", backoffTime)
				time.Sleep(time.Duration(backoffTime) * time.Millisecond)
				backoffTime *= 2
				continue

			// Other errors - Just return the error itself
			default:
				return data, fmt.Errorf( data["error"].(map[string]interface{})["message"].(string) )
		}
	}
	return data, fmt.Errorf("HTTP Error - Retry limit exceeded")
}

{{</ highlight >}}

And, in this case, we will also want to change the main function accordingly, to receive the a `map[string]interface{}` response from the 
`execute_request` function rather than and `http.Response` object. 

{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
func main() {
// ...Previous code unchanged

	// Request with pagination
	for page := 1; true; page++{

		// Execute the request
		fmt.Printf("Extracting page %d\n", page)
		json_data, err := execute_request(req)
		if err != nil {
			log.Fatal(err)
		}
		bytes_data, err := json.Marshal(json_data)
		if err != nil {
			log.Fatal(err)
		}

		// Save the results for this page on GCS
		key = fmt.Sprintf("%d.json", page) // OBS: This key is PURPOSEFULLY bad. Will discuss why later...
		w := bucket.Object(prefix + key).NewWriter(ctx)
		if _, err := w.Write(bytes_data); err != nil {
			log.Println("Error saving page to gcs.")
		}
		if err := w.Close(); err != nil {
			log.Println("Error closing GCS Object writer")
		}

// ...Remaining code unchanged

}
{{</ highlight >}}

And this should be enough to handle any rate limiting issues. Now, the next problem we have to think about is how our program will recover from 
failures.  

## State Management  

When an extraction job is big enough, failures will happen. They may occur because of unhandled edge cases in the incoming data, unavailability of the 
infrastructure we're using or simply because the API host server is down.  

Therefore, it's necessary to anticipate and and plan to recover from such failures. The simplest way do this is by implementing *State Mangement*, AKA 
*Checkpoiting*. Simply put, this about having our program backup information about the current status of execution, that can later be used to resume 
the execution from where it stopped in the case of a crash.  

Most APIs will use cursors for pagination and, also most of the times, state management can be as simple as storing and loading these cursors.


