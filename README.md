# C# Live Project - Code Retrospective

## Table of Contents
* <a href="#introduction">Introduction</a>
* <a href="#mapping">Utilizing an Open Source Map API</a>
* <a href="#pagination">Pagination for User Lists</a>
* <a href="#filtering">Filter Database Results</a>
* <a href="#dashboard">Dashboard Color Scheme</a>
* <a href="#icons">Simplifying Design with Icons</a>
* <a href="#summary">Summary</a>

### <span id="introduction">Introduction</span>
During my C# Live Project at the Tech Academy, I worked for two weeks with a development team of peers on an ASP.NET MVC application. The application was for a construction company with the goal of managing employee scheduling and job site information. For this project, I wrote both front and back end code using HTML, CSS, JavaScript, and C#. We used a code first database with the Entity Framework. I have included example images and code snips of my contributions to the application.

### <span id="mapping">Utilizing an Open Source Map API</span>
Each job site for the construction company has a details page on the application. This page lists the job title, ID number, address, etc. As an added feature to this page, I implemented an open source map API to take the construction site address from the database and return a marked point on the map to show the user. Since the address field is not required for the database, I added a search bar and zoom options to the map so the user can manually find a location if they would like to. The foundation for this feature was the OpenStreetMap API, which was used in conjunction with Leaflet.js for the primary functionality, and Nominatim for Geocoding the address. The JavaScript for mapping is shown below. It also required some HTML, though I left that out for sake of brevity. 

```javascript
function initMap() {
    // Set the address from the database and initialize the starting map view/zoom level
    var address = '@Model.StreetAddress @Model.State @Model.Zipcode';
    var map = L.map('map').setView([0, 0], 14);
    L.tileLayer('http://{s}.tile.osm.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors'
    }).addTo(map);

    // Convert the address into coordinates and map it with a marker
    geocoder = new L.Control.Geocoder.Nominatim();
    geocoder.geocode(address, function (results) {
        latLng = new L.LatLng(results[0].center.lat, results[0].center.lng);
        marker = new L.Marker(latLng);
        map.setView(latLng);
        marker.addTo(map);           
    });
    L.Control.geocoder().addTo(map); // Add a search bar for the user to manually find a location
}

initMap(); // Call the function to create the map
```
Here is a snapshot of what the map looks like from the user's perspective. 
![Map API image](https://github.com/jrs-scott/C-Sharp-Code-Retrospective/blob/master/mapAPI.JPG)

This feature took a large amount of research in order to figure out what resources work best together, and identifying the most efficient way to implement them. I look forward to working with more APIs in the near future. 

### <span id="pagination">Pagination for User Lists</span>
For the admin view of the application, a list of users are shown in two places: a subsection within the main dashboard and a primary page dedicated to managing users. Because the amount of results in the user list will inevitably expand well beyond a full page, I added the ability to page through them. This made it much more manageable for admins to view and interact with the user list. ASP.NET has a built-in pagination helper method named PagedList.MVC which is what I used to achieve this. Below is the relevant C# code within the partial view controller method.

```c#
public ActionResult _UserListPartial(int? page = 1)
{
  ...

	// Variables for pagination
	int pageSize = 10; // Amount of results listed per page
	int pageNumber = page ?? 1; // If pageNumber is null, start the results at page 1

	// Sort registered users by last name descending, then by first name
	var allUsers = db.Users.OrderBy(u => u.LName).ThenBy(u => u.FName);

	// Return the paginated list to the partial view, which is rendered within AllUsers.cshtml
	// If the paging link is clicked on the dashboard, it redirects to AllUsers (with the associated page number)
	return Request.IsAjaxRequest()
		? (ActionResult)PartialView("__UserListPartial", allUsers.ToPagedList(pageNumber, pageSize))
		: PartialView(allUsers.ToPagedList(pageNumber, pageSize));
}
```
After ensuring the controller method was communicating correctly with the cshtml page, here is the result of that partial view, sorted and paged.
![User pagination list](https://github.com/jrs-scott/C-Sharp-Code-Retrospective/blob/master/userPagination.JPG)

Pagination is practical for almost any type of list results, so I am glad to now have experience working with one of the most widely used pagination methods in ASP.NET applications. 

### <span id="filtering">Filter Database Results</span>
Another user story I completed was adding the ability for admins to filter job schedules either by user, or by a weekly date range. This required adding to the index method in the schedule controller, and updating the cshtml page to show dropdown lists with a button to trigger the filter. First, the C# code that returns results after being filtered:

```c#
  [HttpPost]
  public ViewResult Index(string SearchWeek, string Person)
  {   // Filter jobs results by week  
      if (SearchWeek != "")
      {
          string[] weekParts = SearchWeek.Split(' ');
          DateTime weekStart = Convert.ToDateTime(weekParts[0]);
          DateTime weekEnd = Convert.ToDateTime(weekParts[2]).AddDays(1);               
          var jobSearch = (from x in db.Jobs join y in db.Schedules on x.JobId equals y.Job.JobId
                           where (y.StartDate >= weekStart) && (y.StartDate <= weekEnd)
                           || (y.EndDate <= weekEnd) && (y.EndDate >= weekStart)
                           || (y.StartDate <= weekStart) && (y.EndDate >= weekEnd)
                           || (y.StartDate <= weekStart) && (y.EndDate == null)
                           select x).Distinct().ToList();                
          return View(jobSearch);          
      }
      // Filter jobs assigned to the selected user/employee
      else if (Person != "")
      {
          var jobSearch = (from x in db.Jobs
                           join y in db.Schedules on x.JobId equals y.Job.JobId
                           where y.Person.Id == Person && x.Schedules.Count > 0
                           select x).Distinct().ToList();
          return View(jobSearch);
      }
      // Show all scheduled jobs
      else
      {
          var jobs = db.Jobs.Where(j => j.Schedules.Count > 0).ToList();
          return View(jobs);
      }                                 
  }
```

Second, here is the corresponding cshtml code, which calls JavaScript functions to reset the filters when needed.
```cshtml
@using (Html.BeginForm(FormMethod.Post))
{
    <p>
        @* Filters for searching by user, week or all schedules. Default option is "select". Only one filter can have a value at any time - the others will reset *@
        Search by employee: @Html.DropDownList("Person", ((ConstructionNew.Controllers.SchedulesController)this.ViewContext.Controller).GetUsers(), "-- select --",  new { @onclick="filterReset('person')" })
        Or filter by week: @Html.DropDownList("SearchWeek", ((ConstructionNew.Controllers.SchedulesController)this.ViewContext.Controller).ListWeeks(), "-- select --", new { @onclick="filterReset('searchWeek')" })
        <button type="submit">Search</button>
        <button type="button" onclick="resetForm()">View All Schedules</button>
    </p>
}
```

And finally, here are the JavaScript functions that are called by the above on click events.
```javascript
// If search is filtered by week, reset person selection filter. If search is filtered by person, reset searchweek selection.
function filterReset(filterType) {
    if (filterType == 'person') {
        document.getElementById("SearchWeek").selectedIndex = 0;
    } else if (filterType == 'searchWeek') {
        document.getElementById("Person").selectedIndex = 0;
    }
}
// Clears both person and week filter to return all schedules
function resetForm() {
    filterReset('person');
    filterReset('searchWeek');
    document.querySelector("form[action='/Schedules']").submit();
}
```

This feature makes it easy for admins to find what they're looking for - whether that's a specific date range, or a particular employee. 

### <span id="dashboard">Dashboard Color Scheme</span>
My first front end user story was to update the application's color scheme. The colors are from the primary image on the website, so it creates a cohesive, refined look. This task required mostly CSS, with a couple of small HTML changes. A snapshot of the new look:
![Dashboard color palette](https://github.com/jrs-scott/C-Sharp-Code-Retrospective/blob/master/dashboard.JPG)

In order to make the site's color palette easy to update in the future, I used CSS variable for the colors.
```css
:root {
    --dark-brown: #534E54;
    --light-brown: #A79885;
    --light-grey: #F2F2F2;
    --dark-blue: #4E6172;
    --dark-green: #444C13;
    --primary-yellow: #F2C03A;
}
```

This was a fun story to complete since it didn't take long, yet made a large impact on the site's appearance.

### <span id="icons">Simplifying Design with Icons</span>
When an admin is logged in, they have the ability to edit users, job details, company news, etc. For most of those features, there was simple text with links to edit, see details, or delete items. My user story was to update the plain text into easily identifiable icons that still linked to the appropriate resource. 
![Icons for the news page](https://github.com/jrs-scott/C-Sharp-Code-Retrospective/blob/master/newsPage.JPG)

This change made the UI less cluttered while remaining user friendly.

### <span id="summary">Summary</span>
Working on this MVC application was a valuable learning experience. During the two week sprint I...
* Gained a better understanding of team communication and setting priorities
* Expanded my knowledge of C#, JavaScript, HTML using Razor syntax, and CSS
* Worked with an open source API for the first time
* Kept myself on track to complete tasks within a short timeframe
* Refined my ability to research new technologies and evaluate which is best suited for the application
* Version control via Visual Studio
