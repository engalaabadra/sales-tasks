# Sales

### 1. Scenario-Based Automation Design Task
Scenario: Imagine you receive an email attachment in a specific folder, and the attachment is a PDF form containing sales data. Your goal is to create a workflow that extracts information from the PDF and updates a "Sales Records" database (such as an Excel file or SQL table).
Task:
Write down the steps you would take to automate this process.
Identify the types of connectors, actions, or conditions you would use.
Explain how you would handle errors if, for example, the email has no attachment or the PDF is missing required data.

### Steps In Summery : 
    check new emails & save attachments if exist & convert file into text (extracting data from this file) -> to save this data in db 

### Connectors, Actions, and Conditions
- Connectors:  
** Mail Server Connector: To access incoming emails.  
** File Storage (Local/Cloud): To save attachments and processed files.

- Actions:  
File Parsing: Extract text data from the PDF.

- Database Update:  
 Insert or update sales data records.

- Conditions:   
Check for attachment existence.
Validate extracted data fields to ensure completeness.

### Error Handling
No Attachment in Email: Log the error and send a notification to alert administrators.
Invalid Data in PDF: Log the issue, skip the update, and notify relevant parties for manual verification.
Database Error: Log the exception details and notify the team to investigate database-related issues.

### Steps In Details
- Receive the Email:

Use a package like ** Laravel Mailbox ** to capture incoming emails or utilize Laravel's Mail functionality with IMAP to check for new emails.
Configure the mail listener to store attachments in a specific folder within the application.

- Check for Attachments:

As the email is received, check if there are any attachments.
If there is no attachment, log this error and exit the process to avoid unnecessary steps.

- Save Attachment and Extract Text from PDF:

Use a library such as smalot/pdfparser or Spatie/PdfToText to parse the PDF file.
Extract the relevant sales data fields needed to update the database.

- Validate the Extracted Data:

Ensure that the data retrieved from the PDF is in the expected format.
If any required fields are missing, log this error and send an error notification to alert the team that manual intervention is needed.

- Update the Database:

Once validated, map the extracted data fields to the corresponding fields in the "Sales Records" database.
Use Laravel’s Eloquent ORM to insert or update records in a SQL table, or use maatwebsite/excel if an Excel file is used as the data storage.

- Error Logging & Notification:

Log any errors encountered during the process (e.g., missing fields in PDF, invalid data formats).
Send an email or in-app notification to notify administrators or users about errors that require manual handling.

- Schedule & Automate the Workflow:

Use Laravel’s Task Scheduler to automate this workflow at regular intervals to check for new emails or PDF files.

## Examples Implementation
- install package :
```
composer require webklex/php-imap
```
- Configure IMAP Settings in .env file 
```
IMAP_HOST=imap_host
IMAP_PORT=993
IMAP_ENCRYPTION=ssl
IMAP_USERNAME=email@example.com
IMAP_PASSWORD=password
```
#### controller
```
class  ExtractingDataController extends Controller {
    private function getInbox()
    {
        try {
            $clientManager = new ClientManager();
            $client = $clientManager->make([
                'host'          => env('IMAP_HOST'),
                'port'          => env('IMAP_PORT'),
                'encryption'    => env('IMAP_ENCRYPTION'),
                'validate_cert' => true,
                'username'      => env('IMAP_USERNAME'),
                'password'      => env('IMAP_PASSWORD'),
                'protocol'      => 'imap'
            ]);

            $client->connect();
            $inbox = $client->getFolder('INBOX');
            $emails = $inbox->query()->unseen()->get();

            return $emails;

        } catch (\Exception $e) {
            \Log::error('Failed to retrieve inbox emails', ['error' => $e->getMessage()]);
            return null;
        }
    }

    private function extractDataFromText($text)
    {
        // Initialize an empty array to store extracted data
        $data = [];

        // Extract "Amount" value with regex
        if (preg_match('/Amount:\s*\$?([\d,]+(\.\d{2})?)/', $text, $amountMatch)) {
            $data['amount'] = floatval(str_replace(',', '', $amountMatch[1]));
        }

        // Extract "Date" in the format YYYY-MM-DD
        if (preg_match('/Date:\s*([0-9]{4}-[0-9]{2}-[0-9]{2})/', $text, $dateMatch)) {
            $data['date'] = $dateMatch[1];
        }

        // Extract "Client" name (assuming it's text without digits)
        if (preg_match('/Client:\s*([a-zA-Z\s]+)/', $text, $clientMatch)) {
            $data['client'] = trim($clientMatch[1]);
        }

        // Additional fields can be extracted as needed
        // For example, extracting an "Order ID" or "Sales Rep"
        if (preg_match('/Order ID:\s*([A-Za-z0-9\-]+)/', $text, $orderIdMatch)) {
            $data['order_id'] = $orderIdMatch[1];
        }

        if (preg_match('/Sales Rep:\s*([a-zA-Z\s]+)/', $text, $salesRepMatch)) {
            $data['sales_rep'] = trim($salesRepMatch[1]);
        }

        // Log error if any required field is missing
        if (empty($data['amount']) || empty($data['date']) || empty($data['client'])) {
            \Log::error('Missing required fields in extracted data', ['text' => $text]);
        }

        return $data;
    }
    //Step 1: Checking for New Emails with Attachments
    public function checkEmail()
    {
        // Use Laravel Mailbox or an IMAP library to access the inbox
        $inbox = $this->getInbox(); // Receive the Email
        foreach ($inbox->emails as $email) {
            if ($email->hasAttachments()) { // Check for Attachments
                $this->saveAttachment($email->attachment);// Save Attachment
            } else {
                \Log::error('Email has no attachment');
            }
        }
    }
    //Step 2: Parsing and Validating PDF Data
    public function parsePdf($filePath)
    {
        $parser = new \Smalot\PdfParser\Parser();
        $pdf = $parser->parseFile($filePath);
        $text = $pdf->getText();
        
        // Extract fields (this would depend on the PDF format)
        $data = $this->extractDataFromText($text);
        
        if (!$this->isValidData($data)) {
            \Log::error('Invalid data in PDF');
            return false;
        }
        
        return $data;
    }

    

    //Step 3: Updating the Database
    public function updateSalesRecords($data)
    {
        try {
            SalesRecord::updateOrCreate(
                ['id' => $data['id']],
                $data
            );
        } catch (\Exception $e) {
            \Log::error('Failed to update sales records', ['error' => $e->getMessage()]);
        }
    }
}
```

### 2. API Integration Task
Scenario: You have a REST API endpoint that gives a JSON response with a list of employees:
GET https://api.example.com/employees
Example Response:
[
  {
    "id": 1,
    "name": "John Doe",
    "email": "john.doe@example.com",
    "department": "Sales"
  },
  ...
]
Task:
Write a small script (in Python, JavaScript, or a language of your choice) that:
Retrieves the employee list from the API.
Filters employees that belong to the "Sales" department.
Creates a new JSON object containing only name and email of those employees.
Saves this JSON to a local file called sales_employees.json.

### Steps In Summery :
    get data & filter & return data (name , email) & save this data into file

### Steps In Details:
1. Make a GET request to the API to retrieve the employee list.
2. Filter employees belonging to the "Sales" department.
3. Transform the data to keep only the name and email of "Sales" employees.
4. Save the transformed data to a JSON file called sales_employees.json.

### Implementation
```
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Storage;
class fetchSalesEmployeesController extends Controller{
    public function fetchAndSaveSalesEmployees()
    {
        // Step 1: Fetch employees from the API
        $response = Http::get('https://api.example.com/employees');

        if ($response->successful()) {
            $employees = $response->json();
            // Step 2: Filter employees in the "Sales" department
            $salesEmployees = collect($employees)->filter(function ($employee) {
                return $employee['department'] === 'Sales';
            })->map(function ($employee) {
                // Step 3: Transform to include only 'name' and 'email'
                return [
                    'name' => $employee['name'],
                    'email' => $employee['email'],
                ];
            })->values()->toArray();

            // Step 4: Save data to 'sales_employees.json' in local storage
            Storage::disk('local')->put('sales_employees.json', json_encode($salesEmployees, JSON_PRETTY_PRINT));

            return response()->json(['message' => 'Sales employees data saved successfully.']);
        }

        // Handle error if API request fails
        return response()->json(['error' => 'Failed to fetch employees from the API'], 500);
    }
}

```

### 3. Data Transformation Task (Simulating Power BI)
Scenario: You have the following CSV file (provided as text):
ID, Name, Sales, Region
1, John, 2000, North
2, Sarah, 3500, East
3, Ali, 1800, North
4, Mike, 4000, West
5, Emily, 2200, East
Questions:
Write the SQL query to get the total sales per region.
If this was imported into Power BI, how would you create a visualization that shows Sales by Region?
Write a simple formula in DAX to add a new column called Bonus that gives a 5% bonus to every employee whose sales are greater than 2000.

### Steps:
1. SQL Query for Total Sales per Region
```
SELECT Region, SUM(Sales) AS TotalSales
FROM sales_data
GROUP BY Region;
```
this will return the total sales for each region.

2. Power BI Visualization for Sales by Region
In Power BI, to create a visualization that shows Sales by Region, follow these steps:
#### steps :
- Load the CSV File: First, import the CSV file containing the data into Power BI.
- Create a Bar Chart: Use a bar chart to represent sales totals by region.
- Select the Bar Chart visualization.
- Drag the Region column into the axis (X-axis) field.
- Drag Sales into the value field to display the sum of sales per region automatically.
- Format the Visualization:
    Customize labels, colors, and add data labels to show the total sales values above each bar.

This approach gives a clear view of the total sales by region, with each region represented as a separate bar.

3. DAX Formula for a Bonus Column
To add a new column in Power BI that calculates a 5% bonus for employees with sales greater than 2000, use the following DAX formula in the Data view:

```
Bonus = IF(sales_data[Sales] > 2000, sales_data[Sales] * 0.05, 0)
In this formula:
```
This new column, Bonus, will display the calculated bonus values and can be used for further analysis or reporting within Power BI.

### 4. SharePoint List Design & Logic Task
Scenario: Imagine you're working with a SharePoint list that tracks vacation requests from employees. The list contains the following fields:
Employee Name
Start Date
End Date
Status (Pending, Approved, Rejected)
Task:
Write down how you would design a workflow to automatically approve vacation requests if:
The requested number of days is less than or equal to 5.
The employee has no overlapping vacation requests already approved.
Describe how you would handle rejections and notify employees accordingly.

#### Steps In Summery 
    employee can create req vacation & proccess on this operation creation , which is : if num_days_vactions this employee > 5 will reject this req (status = reject) & notify for this employee , if < 5 will approved this request creation & check if this vacation conflict with another vacations for this employee

### Implementation 
Model VacationRequest
```
class VacationRequest extends Model
{
    protected $fillable = ['employee_id', 'start_date', 'end_date', 'status'];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }
}
```
Methods:
```
namespace App\Services;

use App\Models\VacationRequest;
use Carbon\Carbon;

class VacationRequestService
{
    public function processVacationRequest(VacationRequest $request)
    {
        // Calculate number of requested days
        $daysRequested = Carbon::parse($request->start_date)->diffInDays($request->end_date) + 1;

        // Check if request meets automatic approval criteria
        if ($daysRequested <= 5 && !$this->hasOverlappingRequests($request)) {
            $request->update(['status' => 'Approved']);
        } else {
            $request->update(['status' => 'Rejected']);
            $this->notifyEmployee($request);
        }
    }

    private function hasOverlappingRequests(VacationRequest $request)
    {
        return VacationRequest::where('employee_id', $request->employee_id)
            ->where('status', 'Approved')
            ->where(function ($query) use ($request) {
                $query->whereBetween('start_date', [$request->start_date, $request->end_date])
                      ->orWhereBetween('end_date', [$request->start_date, $request->end_date]);
            })
            ->exists();
    }

    private function notifyEmployee(VacationRequest $request)
    {
        // Send notification logic (email, SMS, etc.)
        Mail::to($request->employee->email)->send(new VacationRequestRejected($request));
    }
}
```
Mail 
```
use Illuminate\Mail\Mailable;

class VacationRequestRejected extends Mailable
{
    protected $request;

    public function __construct(VacationRequest $request)
    {
        $this->request = $request;
    }

    public function build()
    {
        return $this->subject('Vacation Request Rejected')
                    ->view('emails.vacation_request_rejected', ['request' => $this->request]);
    }
}
```
contoller
```
namespace App\Http\Controllers;

use App\Models\VacationRequest;
use App\Services\VacationRequestService;

class VacationRequestController extends Controller
{
    protected $service;

    public function __construct(VacationRequestService $service)
    {
        $this->service = $service;
    }

    public function store(Request $request)
    {
        $vacationRequest = VacationRequest::create($request->all());
        $this->service->processVacationRequest($vacationRequest);

        return response()->json(['status' => $vacationRequest->status]);
    }
}
```

### 5. General Programming Problem-Solving Task
Takes an array of numbers as input.
Returns a new array where each element is the product of all the numbers in the original array except the number at that position.
Example: Input: [1, 2, 3, 4]
Output: [24, 12, 8, 6]
Goal: This task assesses your problem-solving skills and your understanding of programming fundamentals.

#### Summery:
    The function takes an array of numbers and returns a new array where each element is the product of all the numbers except the one at the current index .

--- controller
```
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class ArrayController extends Controller
{
    private function calculateProductExceptSelf(array $nums): array
    {
        $length = count($nums);
        $result = array_fill(0, $length, 1);

        $leftProduct = 1;
        for ($i = 0; $i < $length; $i++) {
            $result[$i] = $leftProduct;
            $leftProduct *= $nums[$i];
        }

        $rightProduct = 1;
        for ($i = $length - 1; $i >= 0; $i--) {
            $result[$i] *= $rightProduct;
            $rightProduct *= $nums[$i];
        }

        return $result;
    }
    public function productExceptSelf(Request $request)
    {
        $nums = $request->input('nums'); // Expecting an array of numbers from the request
        $result = $this->calculateProductExceptSelf($nums);
        
        return response()->json($result);
    }
}
```
--- Route
```
Route::post('/product-except-self', [ArrayController::class, 'productExceptSelf']);
```
-- test :
    in this route will enter this array : [1, 2, 3, 4] , will return [24, 12, 8, 6]

### 6. Error Handling and Logging Task
a list of numbers from a file, computes the average, and saves the result to another file. The script should handle the following error scenarios:
The input file is missing.
The input file contains non-numeric data.
The list is empty.
Log each error appropriately to a log file.

### steps :
1. Create an Artisan Command
```
php artisan make:command CalculateAverage
```
2. file CalculateAverage
```
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Storage;

class CalculateAverage extends Command
{
    protected $signature = 'calculate:average';
    protected $description = 'Calculates the average of numbers from a file';

    public function handle()
    {
        $inputFile = 'numbers.txt';       // Path to input file
        $outputFile = 'average.txt';      // Path to output file
        $logFile = storage_path('logs/average_calculation.log');

        try {
            // Check if file exists
            if (!Storage::exists($inputFile)) {
                throw new \Exception("Input file is missing.");
            }

            // Read numbers from file
            $content = Storage::get($inputFile);
            $numbers = array_filter(array_map('trim', explode(PHP_EOL, $content)));

            // Check if list is empty
            if (empty($numbers)) {
                throw new \Exception("The list is empty.");
            }

            // Validate and filter numeric values
            $numericValues = [];
            foreach ($numbers as $number) {
                if (is_numeric($number)) {
                    $numericValues[] = (float) $number;
                } else {
                    Log::warning("Non-numeric data found and ignored: $number", ['context' => 'non-numeric']);
                }
            }

            // If no valid numbers, throw error
            if (empty($numericValues)) {
                throw new \Exception("No valid numeric data in file.");
            }

            // Calculate average
            $average = array_sum($numericValues) / count($numericValues);

            // Write average to output file
            Storage::put($outputFile, "Average: $average");
            $this->info("Average calculated and saved to $outputFile");

        } catch (\Exception $e) {
            // Log any errors to log file
            Log::error($e->getMessage());
            $this->error("Error: " . $e->getMessage());
        }
    }
}
```
in kernel.php
```
protected $commands = [
    \App\Console\Commands\CalculateAverage::class,
];
```
execution
```
php artisan calculate:average
```

### 7. Data Cleanup Task
Scenario: You are given a CSV file with contact information. Unfortunately, the data has some inconsistencies:
Names are sometimes in uppercase, sometimes in lowercase, and sometimes a mix.
Phone numbers are in various formats (e.g., 123-456-7890, (123) 456 7890, 1234567890).
There are missing values in the email column.
Task: Write a Python script that:
Standardizes the names to Title Case.
Normalizes phone numbers to the format 123-456-7890.
Removes any rows that have missing email addresses.
1. create a command :
```
php artisan make:command CleanContactData
```
2. file CleanContactData
```
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Storage;
use League\Csv\Reader;
use League\Csv\Writer;
use Illuminate\Support\Str;

class CleanContactData extends Command
{
    protected $signature = 'contacts:clean';
    protected $description = 'Clean and standardize contact data from CSV';

    public function handle()
    {
        // Load the CSV file from storage
        $path = storage_path('app/contacts.csv');
        if (!file_exists($path)) {
            $this->error('The contacts.csv file does not exist.');
            return;
        }

        // Read the CSV
        $csv = Reader::createFromPath($path, 'r');
        $csv->setHeaderOffset(0);
        $records = $csv->getRecords();

        $cleanedContacts = [];

        foreach ($records as $record) {
            // Standardize name to title case
            $name = Str::title($record['Name']);

            // Normalize phone number to 123-456-7890 format
            $phone = preg_replace('/\D/', '', $record['Phone']);
            if (strlen($phone) === 10) {
                $phone = substr($phone, 0, 3) . '-' . substr($phone, 3, 3) . '-' . substr($phone, 6);
            } else {
                $phone = null; // Invalid phone numbers can be handled or skipped
            }

            // Skip records with missing email
            if (empty($record['Email'])) {
                continue;
            }

            $cleanedContacts[] = [
                'Name' => $name,
                'Phone' => $phone,
                'Email' => $record['Email'],
            ];
        }

        // Write the cleaned data to a new CSV file
        $outputPath = storage_path('app/cleaned_contacts.csv');
        $writer = Writer::createFromPath($outputPath, 'w+');
        $writer->insertOne(['Name', 'Phone', 'Email']);
        $writer->insertAll($cleanedContacts);

        $this->info('Data cleanup complete. Cleaned data saved to cleaned_contacts.csv');
    }
}
```
run 
```
php artisan contacts:clean
```
This will generate cleaned_contacts.csv in the storage/app directory.

### Explanation of Code
Read CSV: Using League\Csv\Reader, the script reads contacts.csv from the storage/app directory.
Standardize Names: Str::title($record['Name']) converts each name to title case.
Normalize Phone Numbers: This uses a regular expression to strip non-digit characters and reformats valid 10-digit phone numbers.
Remove Rows with Missing Emails: The continue statement skips records without an email address.
Write to CSV: The cleaned data is saved to cleaned_contacts.csv.

### 8. Flowchart Task
Scenario: You need to create a flowchart that outlines the process of approving a new project proposal in a company. The process involves:
Initial submission by an employee.
Review by the department head.
Budget approval by finance.
Final approval by the CEO.
Task: Draw a flowchart (you can use any software or draw it by hand and take a picture) that visualizes this process.

1. Flowchart Structure
The flowchart has the following steps:

    - Initial Submission - Employee submits the project proposal.
    - Review by Department Head - The department head reviews the proposal.
    - Budget Approval by Finance - The proposal moves to the finance department for budget approval.
    - Final Approval by CEO 
    - Once budget is approved, the proposal moves to the CEO for final approval.
    Each step includes decision points (e.g., whether to approve or reject), where different paths are followed based on the decision.

2. Implementing the Flowchart via -> Draw.io , Hand-drwan sketch
    ```
    Start
    ↓
    [Employee Submits Proposal]
    ↓
    [Department Head Review]
    ↓ Yes
    [Approved?] -----> No -----> [Rejected - Notify Employee]
    ↓
    [Finance Review]
    ↓ Yes
    [Budget Approved?] -----> No -----> [Rejected - Notify Employee]
    ↓
    [CEO Final Approval]
    ↓ Yes
    [Final Approval Granted] -----> No -----> [Rejected - Notify Employee]
    ↓
    End
    ```
### 9. Conditional Logic Task
    Task: Write pseudocode that represents the following logic:
    If a customer has been a member for more than 5 years and has made purchases over $10,000, they receive a Gold membership.
    If a customer has been a member for 3-5 years and has made purchases over $5,000, they receive a Silver membership.
    All other members receive a Bronze membership.

#### Code
```
// Pseudocode for assigning membership level in Laravel

function assignMembershipLevel($customer) {
    // Calculate membership duration in years
    $membershipDuration = calculateYearsSince($customer->membership_start_date);

    // Get total purchase amount
    $totalPurchases = $customer->total_purchases;

    // Determine membership level based on conditions
    if ($membershipDuration > 5 && $totalPurchases > 10000) {
        $customer->membership_level = 'Gold';
    } elseif ($membershipDuration >= 3 && $membershipDuration <= 5 && $totalPurchases > 5000) {
        $customer->membership_level = 'Silver';
    } else {
        $customer->membership_level = 'Bronze';
    }

    // Save the membership level to the database
    $customer->save();
}

// Helper function to calculate the duration in years
function calculateYearsSince($startDate) {
    return now()->diffInYears($startDate);
}
```

#### Explanation of Logic
1. Calculate Membership Duration: Determines how many years the customer has been a member by using the calculateYearsSince helper function.
2. Check Total Purchases: Compares the total purchases against the threshold values.
Assign Membership Level:
3. Assign Gold if the customer has been a member for more than 5 years and has purchases over $10,000.
Assign Silver if the customer has been a member for 3-5 years and has purchases over $5,000.
Assign Bronze for all other cases.

    This pseudocode can be converted to an actual method in a Laravel model or service class where $customer would be an instance of a Customer model.

### 10. SQL Data Extraction Task
Scenario: You have a table called Orders with the following columns: OrderID, CustomerID, OrderDate, TotalAmount.
Task:
Write an SQL query to extract all orders made by a specific CustomerID between 2023-01-01 and 2023-12-31.
Write another query to calculate the total revenue generated by all customers during the same period.
Goal: This task will help us evaluate your ability to write SQL queries to extract and aggregate data, which is a common requirement in data analysis and reporting.

Task 1: Extract Orders for a Specific CustomerID within a Date Range
To retrieve all orders for a specific CustomerID between 2023-01-01 and 2023-12-31, you can use the following SQL query. This example uses a query in Laravel's query builder.

```
use Illuminate\Support\Facades\DB;

$customerId = 123; // specify the desired CustomerID

$orders = DB::table('orders')
    ->where('CustomerID', $customerId)
    ->whereBetween('OrderDate', ['2023-01-01', '2023-12-31'])
    ->get();
```
This will retrieve all rows for the specified CustomerID in the date range provided.

Task 2: Calculate Total Revenue for All Customers in the Same Period
To get the total revenue generated by all customers within the same date range, you can use an aggregate SQL query.
```
$totalRevenue = DB::table('orders')
    ->whereBetween('OrderDate', ['2023-01-01', '2023-12-31'])
    ->sum('TotalAmount');
```
This will return the sum of TotalAmount for all orders within the specified date range, which represents the total revenue for that period.

### Explanation of the Approach
1. Laravel Query Builder: Using whereBetween helps specify the range condition on OrderDate.
2. Aggregate Functions: sum is used to calculate the total revenue from the TotalAmount column.
    
    This approach ensures the data extraction and aggregation tasks are both readable and efficient within the Laravel framework.

