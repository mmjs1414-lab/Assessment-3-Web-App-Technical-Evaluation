# Assessment-3-Web-App-Technical-Evaluation

![LobbyArch screenshot](LobbyArch%20Screenshot.png)

## Introduction 

I developed LobbyArch to help users discover Australian films through the National Film and Sound Archive's (NFSA) lobby card collection. I wanted to address the lack of visual appeal and simplicity in traditional collection interfaces by presenting lobby cards in a visually engaging, easy-to-navigate way. I also wanted users to explore films through related production year and genre without making the browsing experience overwhelming.

LobbyArch retrieves lobby cards from the NFSA API and stores the results in the allResults array. The main carousel displays three cards at a time, with the selected card in the middle. Its production year and genres are then used to find matching cards in the More Like This section. I included a title search engine to make it easier to find specific films without navigating through the entire carousel.

My main technical challenges involved retrieving all API results, developing linked carousels and managing how the interface responds to searches.

## API Pagination and Filtering 

I started by using getData() to request lobby cards from the NFSA API. At that stage, I mainly focused on understanding the JSON structure and displaying the returned images. 

The first limitation I encountered was that the API returned only 25 results per request. This became a problem once I built the carousel and More Like This, because users were only browsing a small portion of the collection.

I changed the approach by introducing **`loadAllResults()`**, which requests successive pages and stores the filtered results in one array.

```javascript
let page = 1;

while (true) {
  const response = await fetch(
    `${currentQueryUrl}&page=${page}&limit=${limit}`
  );

  if (!response.ok) {
    throw new Error("Could not load data from the NFSA API");
  }

  const data = await response.json();
```

The **`while (true)`** loop continues requesting pages until the stopping condition is reached. I used **`async/await`** because the application needs to wait for each API response before processing it. The **`response.ok`** condition checks whether the request succeeded before converting the response to JSON.

```javascript
const filteredResults = data.results.filter(item => {
  const genres = item.parentTitle.genres;
  if (genres === null) {
    return true;
  }
});

allResults = allResults.concat(filteredResults);

if (data.results.length < limit) {
  break;
}
page = page + 1;
```

The **`null`** condition was important because excluding every record without a reported genre would unnecessarily reduce the collection. I used **`concat()`** to combine each page's filtered results with the results already stored. 

I decided to load all the results at the start so users could browse the full collection through the carousels and search. I could have loaded 25 results at a time as users explored more cards, which would have made the initial loading faster. However, I would then need to change how the carousels and search access the results.

I used ChatGPT when I was trying to work out how to fetch more than 25 results from the API. It helped me understand how a **`while loop`** could be used in **`loadAllResults()`** to keep requesting pages until there were no more results. I checked the browser console while testing the code to confirm that allResults contained results from different pages, not just the first 25. One issue with this approach is the loading time. I added a "Loading..." message because the page would otherwise appear empty while the requests were running. 

This was one of the most useful changes I made because it allowed the different sections of LobbyArch to use the same data collection.

## Carousel Development and Related Results

My original approach was to use **`.forEach()`** to display multiple lobby cards. I later replaced this with a carousel because I wanted the application to follow my prototype and offer a browsing experience different from a typical collection website.

The carousel needed to display three cards while keeping the selected card in the middle. I introduced **`middleImage`** to store its position and calculate the side results.

```javascript
let previousIndex = middleImage - 1;
let nextIndex = middleImage + 1;

if (previousIndex < 0) {
  previousIndex = results.length - 1;
}

if (nextIndex >= results.length) {
  nextIndex = 0;
}

const previousItem = results[previousIndex];
const item = results[middleImage];
const nextItem = results[nextIndex];
```

The conditions allow the carousel to loop, preventing it from stopping at the first or last result. I also added a random initial middleImage value so that refreshing the page could lead users to a different lobby card.

I had to keep track of **`middleImage`** each time users clicked the previous or next button so the correct card would appear in the centre. Displaying the cards in a grid would have required less JavaScript, but I kept the carousel to stay close to my original prototype.

In the More Like This section, I used **`.filter()`** to find lobby cards produced in the same year as the selected card. The genre results were harder to filter because some **`parentTitle.genres`** arrays contain multiple genres. I initially tried to compare the genre values using **`===`**, but that caused an error because it wouldn’t identify cards that shared only one genre.

```javascript
if (currentGenre === null) {
  sameGenreResults = allResults.filter(film => {
    return film.parentTitle.genres === null;
  });
} else {
  sameGenreResults = allResults.filter(film => {
    return Array.isArray(film.parentTitle.genres) &&
    film.parentTitle.genres.some(genre =>
      currentGenre.includes(genre)
    );
  });
}
```
I used **`Array.isArray()`** to check whether **`parentTitle.genres`** contained an array before comparing genres. The **`.some()`** and **`.includes()`** methods then check whether any genre matches one from the selected lobby card, rather than comparing the entire arrays. I also added a separate condition for **`null`** values so that cards without genre information could still appear alongside other cards with missing genres.

ChatGPT helped me fix the genre filtering when More Like This displayed results that didn't match the selected card. It suggested using **`Array.isArray()`** to check whether the genre data was an array, then using **`.some()`** and **`.includes()`** instead of **`===`** to identify cards that shared at least one genre. To make the code less DRY, I relied on ChatGPT to reduce repeated HTML by entirely developing **`displayCards()`** to be reused for both the Same Year and Same Genre carousels.

## Search Functionality and Carousel State

I added search after developing the main browsing features. The purpose was to let users find a specific lobby card if they already knew its title, since it would be impractical to browse the large carousel to find a single result. 

I used **`.filter()`** to search for matching titles in **`allResults.`**. The **`searchInput.addEventListener("input", ...)`** detects changes in the search field and updates the main carousel as users type. I could have sent a new API request for each search, but I kept the filtering within JavaScript since the data was already available.

```javascript
searchInput.addEventListener("input", function () {
  const searchTerm = searchInput.value.toLowerCase();

  const searchResults = allResults.filter(item => {
    return item.title &&
    item.title.toLowerCase().includes(searchTerm);
  });

  middleImage = 0;

  if (searchResults.length > 0) {
    displayResults(searchResults, "hero");
  } else {
```

I used **`.toLowerCase()`** to make the comparison case-insensitive and **`.includes()`** to match part of a title. The **`item.title`** condition also checks that a title exists before applying the string method.

While working on the search function, I noticed that **`middleImage`** kept the index of the previously displayed card. If it was set to 3 and a search returned only two matches, the function would look for a card at an index that didn't exist and return undefined. I added **`middleImage = 0`** to the search function to start each new set of results from the first card.

I added a ‘No results found for [search input]’ message for undetected searches. The More Like This section also needed to be cleared, since it was still displaying cards related to the previous selection. 

I had difficulty getting the search function to work with the existing carousel. ChatGPT initially suggested ways to filter titles, but I still had problems with middleImage keeping its previous index and More Like This displaying cards from an earlier selection. I asked for help with these specific issues and how to adjust the suggestions without changing the rest of my code. I also referred to GeeksforGeeks tutorials to understand how the input event listener works and how it could update the results as users type because I wanted to understand the logic. 

## Weaknesses and Technical Debt

One of LobbyArch's biggest weaknesses is its carousel-based structure. I wanted to stay close to my original prototype and create something different from the NFSA collection interface. However, carousels are less practical when users want to view matching results together and choose which cards to inspect.

I noticed repeated lobby cards in the More Like This section. For example, Wake in Fright appeared in all of the main carousel, Same Year, and Same Genre. Both related carousels use **`.filter()`** on **`allResults`** independently, so the selected card can appear in either section, and the results can overlap. I would change the filtering to exclude the selected card and check which cards have already appeared in Same Year before displaying Same Genre.

In addition, the search code has its limitations, as **`.filter()`** and **`.includes()`** only match titles. I would expand it to include production years, genres and keywords from descriptions. I would also add a side menu showing recently viewed cards so users could return to and compare previous selections, along with more information about the selected film.

I overlooked basic HTML elements such as the **`<title>`** tag because I focused mostly on JavaScript. An HTML validator would have helped me identify these issues. I was also satisfied with how the interface looked on my screen until I noticed that elements such as the carousel arrow buttons appeared differently on other screens. I would fix this with CSS and pay more attention to testing across different screen sizes.

## Conclusion 

LobbyArch progressed through design decisions and problem-solving, improving my understanding of JavaScript, API data and error handling. If I continued the project, I would spend more time evaluating the browsing structure to improve usability and functionality. Another priority would be following industry-standard coding practices and reviewing GenAI-generated code to better fit my application.

## References 

GeeksforGeeks. (2025a, July 23). How to implement search filter functionality in React JS? https://www.geeksforgeeks.org/reactjs/how-to-implement-search-filter-functionality-in-reactjs/

GeeksforGeeks. (2025b, August 5). How to filter an array of objects based on multiple properties in JavaScript? https://www.geeksforgeeks.org/javascript/how-to-filter-an-array-of-objects-based-on-multiple-properties-in-javascript/

GeeksforGeeks. (2026a, March 18). Building a carousel with vanilla JavaScript. https://www.geeksforgeeks.org/html/building-a-carousel-with-vanilla-javascript/

GeeksforGeeks. (2026b, June 1). Generate random number in given range using JavaScript. https://www.geeksforgeeks.org/javascript/how-to-generate-random-number-in-given-range-using-javascript/

OpenAI. (2025, August 21). High school grammar concepts [Generative AI chat]. ChatGPT. https://chatgpt.com/share/68a77b60-0ee4-800c-9acc-cd3fd573c311

Tailwind CSS. (n.d.). Styling with utility classes. Retrieved October 9, 2026, from https://tailwindcss.com/docs/styling-with-utility-classes
