---
title: "Developing API data extraction jobs with Go - The very minimum"
series: "Developing API data extraction jobs with Go"
publishDate: 2023-08-28:12:00-03:00
date: 2023-08-19:00:00-03:00
draft: true
---
## Intro

This post is the first installment of a series on building ETL/ELT jobs to perform full data extraction of API data.  
The ultimate goal is to explore the main challenges of building this type of application and the main strategies available to solving them.  

All the programming samples will be done with the Go programming language. However, these same strategies will also
be applicable to all programming languages.

### Parts
Part 1 - Rate Limiting and State Management  
Part 2 - Deployment Options (*Coming soon*)


### What makes an extraction job production ready?

**At a minimum**, for a production environment, a data extraction job should achieve at least these 3 things:
1. **State Management**. For an extraction job that will take hours to complete, it is *not viable* having to restart it every 
time an unexpected crash happens. The program should be able to store information about it's execution state and use it to recover 
from failures.

2. **Logging and monitoring**. For a large enough amount of data, failures are *guaranteed* to happen. There should be execution logs 
that facilitate identifying, quantifying and debbugging these failures.

3. **Retrial mechanisms**. Besides just identifying failures, the job should have the capability to retry 
failed extraction steps.

**Idempotency is desirable, but not required**, which means that running a part of the extraction multiple times will
have the same end result as running it only once on wherever you're storing your data. In the data world, 
not having idempotency will mean having duplicate data in your destination. You can handle deduplication downstream, but it is 
nice having it handled at the extraction step.

**Do not worry about performance initially**. In most cases, reliability and resilience are more important than speed.
Your engineering hours are more valuables than some VM-hours. So, don't feel tempted to mess with async stuff if the
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

Therefore, it's necessary to anticipate and and plan for failures. The simplest way do this is by implementing *State Mangement*, AKA 
*Checkpointing*. Simply put, this is about having our program backup information about the current status of execution that can later be reloaded to resume 
the execution from where it stopped in the case of a crash.  

In our example, the information we need to save is the `after` cursor and the page number. We can begin by declarying a a `state` type and 
function to load and save the state to GCS.

> **Important**: We should avoid storing the `next` URL, because that will contain the access_token in the query parameters.

{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
type state struct {
	After string `json:"after"`
	PageNumber int `json:"page_number"`
}

func saveStateToGCS(ctx context.Context, bucket *storage.BucketHandle, stateObj state) error {
    stateJSON, err := json.Marshal(stateObj)
    if err != nil {
        return err
    }

    w := bucket.Object("_state.json").NewWriter(ctx)
    _, err = w.Write(stateJSON)
    if err != nil {
        return err
    }
    if err := w.Close(); err != nil {
        return err
    }
    return nil
}

func loadStateFromGCS(ctx context.Context, bucket *storage.BucketHandle) (state, error) {
    r, err := bucket.Object("_state.json").NewReader(ctx)
    if err != nil {
        return state{}, err
    }
    defer r.Close()

    var stateObj state
    err = json.NewDecoder(r).Decode(&stateObj)
    if err != nil {
        return state{}, err
    }
    return stateObj, nil
}
}
{{</ highlight >}}

Next, we modify the main function to load the state at the beginning, and save the state to GCS every 100 page. This way, if the execution is interrupted for any reason, we only lose the last 100 pages 
of extraction runtime instead of potential hours.  

{{< highlight go "linenos=false,hl_lines=8 15-17,linenostart=199" >}}
func main() {

	// ...Existing code

	// Load state from GCS
    stateObj, err := loadStateFromGCS(ctx, bucket)
    if err != nil {
        fmt.Println("State not found. Starting from scratch.")
        stateObj = state{ PageNumber: 1, After: ""}
    } else {
        fmt.Printf("Resuming extraction from page %d\n", stateObj.PageNumber)
    }

	// Add the after param if necessary
	if stateObj.After != "" {
		params.Set("after", stateObj.After)
	}

	// ...Existing code

	// Request with pagination
	for page := stateObj.PageNumber; true; page++{

		// ...Existing code

		// Update and save the state
		stateObj.PageNumber = page + 1
		stateObj.After = json_data["paging"].(map[string]interface{})["cursors"].(map[string]interface{})["after"].(string)
		if page%100 == 0 {
            if err := saveStateToGCS(ctx, bucket, stateObj); err != nil {
                log.Println("Error saving state to GCS.")
            }
        }

	}
	fmt.Println("Extraction ended.")
}
{{</ highlight >}}

