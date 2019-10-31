# 100 Polish novels annotated

An annotated version of [this collection](https://github.com/computationalstylistics/100_polish_novels) of 100 novels in Polish. 

The novels were annotated (lemmas, POS, NER) using the infrastructure provided by the CLARIN-PL consortium, via API. The following R script was used to complete the task:


``` R

library(httr)
library(rjson)

# set the directory containing your input files
in_path = "ner_in"
# set the output directory
out_path = "ner_out"

# which tool should be involved?
toolname = "liner2"
# for NER, which model?
nermodel = "5nam"

# "names"- granice nazw
# "n82" - 82 kategorie
# "top9" - 9 kategorii
# "5nam" - 5 kategorii
# "timex1" - model timex1
# "timex4" - modle timex4
# "all" - grupa wszystkich modeli (oznaczanych jednocześnie)



# the API url
url = "http://ws.clarin-pl.eu/nlprest2/base" 


# a function for uploading files from harddrive;
# the argument is the filename (as string)
upload = function(file) {
    # send a POST request, the file being attached
    # (description of the API: http://nlp.pwr.wroc.pl/redmine/projects/nlprest2/wiki/UploadApi)
    request = POST(url = paste(url, "/upload/", sep=""), body = upload_file(file, type = "binary/octet-stream"))
    # from the response, get the task's ID as assigned by the server:
    fileid = content(request, type="text", encoding = "UTF-8")
    return(fileid)
}


# list the filenames in the specified 'in_path':
files_to_be_processed = list.files(path = in_path)

# start timer
global_time = unclass(Sys.time())

# blank line on screen:
message(" ")


# iterate over the files 
for(file in files_to_be_processed) {
    
    # reset the timer
    local_time = unclass(Sys.time())
    message(file)
    message("\tProcessing...")
    
    # upload the current file using the function defined above:
    fileid = upload(paste(in_path, file, sep="/"))  # separator in Linux & Mac only
    
    # prepare a JSON query, containig the file ID, tool and model:
    data = list(file = fileid, tool = toolname, model = nermodel)
    doc = toJSON(data)
    
    # send the JSON request, in order to start processing the file on the server
    taskid = content(POST(url = paste(url, "/startTask/", sep=""), body = doc, encode = "json"))
    
    # check whether the task is completed, via a GET request:
    resp = GET(paste(url, "/getStatus/", taskid, sep=""))
    # extract relevant information from the request
    data = fromJSON(content(resp, "text", encoding = "UTF-8"))
    
    # test if the status is "DONE", ...
    while(data$status == "QUEUE" | data$status == "PROCESSING") {
        # ... otherwise, wait 1 second and repeat the request
        Sys.sleep(1)
        resp = GET(paste(url, "/getStatus/", taskid, sep=""))
        data = fromJSON(content(resp, "text", encoding = "UTF-8"))
    }
    # if the status is "ERROR", then report it
    if(data$status == "ERROR") {
        message("\t##### An error occurred! #####")
    }
    if(data$status == "DONE") {
        message("\tDownloading the results...")
        # extract a new file ID from the above request
        data = data$value[[1]]$fileID
        # now, send another request, using the obtained new file ID
        results = GET(paste(url, "/download", data, sep=""))
        # extract the content
        content = content(results, "text", encoding = "UTF-8")
        # save the content in the indicated output directory 'out_path':
        cat(content, file = paste(out_path, "/", file, ".ccl", sep=""), sep="\n")
        message("\tDone in ", round(unclass(Sys.time() - local_time)), " seconds, the tool used: '", toolname, "'")
    }

}

# say something nice when you're done:
message(" ")
message("Operation complete! (elapsed time: ", round(unclass(Sys.time() - global_time)), " seconds)")  

```