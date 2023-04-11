
# Bulk Import (CSV/Excel) using Corn Job - Command line.

We used here -
- API
- JWT Auth
- Corn Job
- Queue
- Bus
- Command Line

------------------------------------------------

## API Lists

All API were in : `routes/api.php`
```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\V1\Admin\Auth\AuthController;
use App\Http\Controllers\V1\Admin\Contact\ContactController;

Route::group(['prefix' => 'auth', 'middleware' => ['api']], function(){

    Route::post('login', [AuthController::class, 'login']);
    Route::post('registration', [AuthController::class, 'registration']);
    Route::post('logout', [AuthController::class, 'logout']);
    Route::post('refresh', [AuthController::class, 'refresh']);
    Route::post('me', [AuthController::class, 'me']);
    Route::post('upload', [ContactController::class, 'store']);
    
    Route::group(['prefix' => 'contact'], function(){
        
        Route::post('parse-import', [ContactController::class, 'parseImport']);
        Route::post('process-import', [ContactController::class, 'processImports']);
       
    });

});

```

We have two Controllers here. One for JWT Auth another for Contact Uploads :

Code in : AuthController.php

```php 
<?php

namespace App\Http\Controllers\V1\Admin\Auth;

use App\Helpers\Helper;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use App\Http\Controllers\Controller;
use Illuminate\Support\Facades\Validator;
use Symfony\Component\HttpFoundation\Response;
use App\Contracts\Services\AuthContract;

class AuthController extends Controller
{
    private AuthContract $authContract;

    public function __construct(AuthContract $authContract)
    {
        $this->authContract = $authContract;
        $this->middleware('auth:api', ['except' => ['login', 'registration']]);
    }


    public function login(Request $request)
    {
        try {
            $validator = Validator::make($request->all(), [
                'email' => 'required|email',
                'password' => 'required|min:6'
            ]);

            if ($validator->fails()) {

                return response()->json(
                    Helper::RETURN_ERROR_FORMAT(Response::HTTP_BAD_REQUEST, 'Validation Error', $validator->errors()->toArray())
                );
            }

            $credentials = $request->only('email', 'password');
            $token = $this->guard('api')->attempt($credentials);

            if($token){
                $bearer_token = $this->respondWithToken($token);

                return response()->json([
                    // Helper::RETURN_SUCCESS_FORMAT(Response::HTTP_OK, 'Login Successfully', $bearer_token)
                    // 'status' => 'success',
                    'data' => $bearer_token
                ]);
            }
        } 
        catch (\Exception $exception) {
            return response()->json(
                Helper::RETURN_ERROR_FORMAT(Response::HTTP_UNAUTHORIZED, 'Invalid Email / Password')
            );
        }
    }

    public function registration(Request $request)
    {
        try{

            $validator = Validator::make($request->all(), [
                'first_name' => 'required',
                'last_name' => 'required',
                'email' => 'required|email',
                'phone' => 'required|numeric',
                'password' => 'required|min:6',
            ]);

            if ($validator->fails()) {

                return response()->json(

                    Helper::RETURN_ERROR_FORMAT(Response::HTTP_BAD_REQUEST, 'Validation Error', $validator->errors()->toArray())
                );
            }

            $response = $this->authContract->userRegistration($request);

                return response()->json(

                    Helper::RETURN_SUCCESS_FORMAT(Response::HTTP_OK, 'User Information Saved!', $response)   
                );

        }
        catch(\Exception $e)
        {
            return response()->json(
                
                Helper::RETURN_ERROR_FORMAT(Response::HTTP_UNAUTHORIZED, 'Registration Failed')
            );
        }
    }


    public function me()
    {
        try{

            $user_info = $this->guard()->user();

            return response()->json(

                Helper::RETURN_SUCCESS_FORMAT(Response::HTTP_OK, 'User Information', $user_info)
            );
        }
        catch (\Exception $e) {

            return response()->json(Helper::RETURN_ERROR_FORMAT(Response::HTTP_UNAUTHORIZED, 'Invalid Email / Password'));
        }
    }


    public function logout()
    {
        try{
            $this->guard('api')->logout();

            return response()->json(
                Helper::RETURN_SUCCESS_FORMAT(Response::HTTP_OK, 'Successfully logged out')
            );
        }
        catch(\Exception $e){

            return response()->json(
                Helper::RETURN_ERROR_FORMAT(Response::HTTP_FORBIDDEN, 'Something went wrong!')
            );
        }
    }


    public function refresh()
    {
        return $this->respondWithToken($this->guard()->refresh());
    }


    protected function respondWithToken($token)
    {
        return response()->json([
            'access_token' => $token,
            'token_type' => 'bearer',
            'expires_in' => $this->guard()->factory()->getTTL() * 60
        ]);
    }

    public function guard()
    {
        return Auth::guard('api');
    }
}

```


Code in : ContactController.php

```php
<?php

namespace App\Http\Controllers\V1\Admin\Contact;

use App\Http\Controllers\Controller;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use App\Contracts\Services\ContactUploadContract;
use App\Helpers\Helper;
use App\Jobs\EmployeeCSVProcess;
use App\Models\Contact;
use App\Models\CSVdata;
use App\Models\Employee;
use Exception;
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\File;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Validator;
use Maatwebsite\Excel\Facades\Excel;

class ContactController extends Controller
{
    // private ContactUploadContract $contactUploadContract;

    // public function __construct(ContactUploadContract $contactUploadContract)
    // {
    //     $this->contactUploadContract = $contactUploadContract;    
    // }


    
    public function store(Request $request)
    {
        try{
            $validator = Validator::make($request->all(), [
                'file' => 'required',
            ]);

            if ($validator->fails()) {

                return response()->json(
                    Helper::RETURN_ERROR_FORMAT(Response::HTTP_BAD_REQUEST, 'Validation Error', $validator->errors()->toArray())
                );
            }

            if(!empty($request->has('file')))
            {
                // if(File::exists(storage_path('public/uploads'.request()->file))){
                //     File::delete(storage_path('public/uploads'.request()->file));
                // }
                // $file = $request->file('file');
                // $fileCustomName = rand().'.'.$file->getClientOriginalExtension();
                // $file->storeAs('public/uploads', $fileCustomName);


                //---------------------------------------------------------------------
                // $data = array_map('str_getcsv', file(request()->file));


                $data = file(request()->file);
                
                // array chunk
                $chunks = array_chunk($data, 200);
                $path = storage_path("app/public/uploads");

                foreach($chunks as $key => $chunk)
                {
                    $name = "/temp{$key}.csv";
                    file_put_contents($path . $name, $chunk);
                }

                // $files = glob("$path/*.csv");
                // $header =[];
                // $batch = Bus::batch([])->dispatch();

                // foreach($files as $key => $file)
                // {
                //     $data = array_map('str_getcsv', file($file));

                //     if($key == 0)
                //     {
                //         $header = $data[0];
                //         unset($data[0]);
                //     }
                //     $batch->add(new EmployeeCSVProcess($data, $header));
                //     unlink($file);
                // }

                return 'File Stored Successfully';           

            }
            
        }
        catch(Exception $e)
        {

        }
        
    }

    public function parseImport(Request $request)
    {
        $path = $request->file('csv_file')->getRealPath();
        $data = array_map('str_getcsv', file($path));

        foreach($data as $key => $value)
        {
            if($key == 0)
            {
                $header = $value;
            }
        }

        if (count($data) > 0) {
            $csv_data = array_slice($data, 0, 2);

            $csv_data_file = CSVdata::create([
                'csv_filename' => $request->file('csv_file')->getClientOriginalName(),
                'csv_header' => json_encode($header),
                'csv_data' => json_encode($data)
            ]);
        }

        return response()->json([

            'status' => 'success',
            'csv_data' => $csv_data,
            'csv_data_file' => $csv_data_file,
        ]);
    }

    public function processImports(Request $request)
    {
        $data = CsvData::find($request->csv_data_file_id);
        $csv_data = json_decode($data->csv_data, true);
        dd($csv_data);
        foreach ($csv_data as $row) {
            $contact = new Contact();
            foreach (config('app.db_fields') as $index => $field) {
                // if ($data->csv_header) {
                //     $contact->$field = $row[$request->fields[$field]];
                // } else {

                    $contact->$field = $row[$request->fields[$index]];

                // }
            }
            $contact->save();
    }
    }
    
}

```

Then we will run the command line : `php artisan contact:upload`. Where we declear all the functionality based on the command in the `handle()` method.
Path : `app/console/commands/ContactProcess.php`

```php
<?php

namespace App\Console\Commands;

use App\Jobs\EmployeeCSVProcess;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Bus;

class ContactProcess extends Command
{

    protected $signature = 'contact:upload';


    protected $description = 'Contact Upload Command';


    public function handle(): void
    {
        $path = storage_path("app/public/uploads");
        $files = glob("$path/*.csv");
        $header =[];

        $batch = Bus::batch([])->dispatch();

        foreach($files as $key => $file)
        {
            $data = array_map('str_getcsv', file($file));

            if($key == 0)
            {
                $header = $data[0];
                unset($data[0]);
            }
            $batch->add(new EmployeeCSVProcess($data, $header));
            unlink($file);
        }
        Artisan::call('queue:work', []);
    }
}

```

Here we declare our Queue of work. We push the work in a queue so that the work process continues simultaneously.

Queue Path : `app/Jobs/EmployeeCSVProcess.php`

```php
<?php

namespace App\Jobs;

use App\Models\Employee;
use App\Models\Test;
use Exception;
use Illuminate\Bus\Batchable;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class EmployeeCSVProcess implements ShouldQueue
{
    use  Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $data;
    public $header;

    public function __construct($data, $header)
    {
        $this->data = $data;
        $this->header = $header;
    }

    /**
     * Execute the job.
     */
    public function handle()
    {
        // try{
        foreach($this->data as $employee)
        {
            $employee_data = array_combine($this->header ,$employee);
            // dd($employee_data);
            Employee::create($employee_data);
        }
    // }
        // catch(Exception $e)
        // {
        //     Log::error('failed', [$e->getMessage(), $e->getLine()]);
        // }  
    }
}

```


