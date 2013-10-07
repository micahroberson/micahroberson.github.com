---
publish: true
layout: post
title: "Upload files directly to S3 w/ Backbone on Heroku"
date: 2013-10-04 09:40
comments: true
keywords: s3, backbone, upload, files, input type file, rails, CORS, Heroku
description: How to upload files directly to an S3 bucket via javascript/coffeescript from a Rails app hosted on Heroku.
cover: 
---

[S3](http://aws.amazon.com/s3/) has been the defacto standard for some time now when it comes to cloud storage, specifically for handling user uploads within an app. PaaS providers often have limitations on storage and/or file uploads - most notably Heroku, which doesn't allow any file writes outside of the /tmp directory. Fortunately this is an easy enough problem to solve, with help from [Paperclip](https://github.com/thoughtbot/paperclip) and [CarrierWave](https://github.com/carrierwaveuploader/carrierwave). Although utilizing one of these options may be easy and not all that different from how you would handle user uploads in a classic Rails app where you CAN write to the filesystem, they aren't all that efficient on Heroku. A simple file upload becomes a data-intesive, long-drawn-out request in order to get the file from the browser(client) to the server(Heroku) and then to S3. Ideally these gems would process the file(s) in the background, and we could say the time to do so is negligible as a result. But when we decide we want to show progress to the user and also know when the upload is complete, things get a lot more complicated!

<!--more-->

This solution was originally built by Rok Krulec at [Dubjoy](http://dubjoy.com), and can be found here: [http://codeartists.com/post/36892733572/how-to-directly-upload-files-to-amazon-s3-from-your](http://codeartists.com/post/36892733572/how-to-directly-upload-files-to-amazon-s3-from-your), but I've forked the code and made some tweaks to handle multiple files, progress, and cancellations.

In order to implement the entirety of this setup, you'll need a CORS enabled bucket. Follow [Step 1](http://codeartists.com/post/36892733572/how-to-directly-upload-files-to-amazon-s3-from-your) for instructions. You'll also need to setup server-side request signing, which Rok Krulec shows how to do in [Step 3](http://codeartists.com/post/36892733572/how-to-directly-upload-files-to-amazon-s3-from-your). 

For the client-side implementation, which is Step 2 from the blog post linked above, we'll do things a little differently.

You can find my forked repo with the necessary S3Upload lib here: [https://github.com/micahroberson/s3upload-coffee-javascript](https://github.com/micahroberson/s3upload-coffee-javascript) 

To a Backbone view I've added the following:

{% codeblock lang:coffeescript %}
  # Listener to detect when the user has selected a file.
  # We could also opt for a click event on an 'Upload Files' button instead
  events:
    'change input#Files': 'onChangeFiles'

  ...

  # Listen for newly added models and render a view for each
  initialize: () ->
    @listenTo @attachmentsCollection, 'add', @onAddAttachment
  ...

  # Instantiate and render new views for models added to the collection
  # This is the view that will be listening to the 'upload:progress' event, and can also allow the user to cancel the upload
  onAddAttachment: (model, response) ->
    attachmentView = new Doozie.Views.Attachments.Show(model: model)
    @$('.card-attachments').append(attachmentView.render().el)
    @attachmentViews[model.cid] = attachmentView

  ...

  # Callback bound above which triggers the actual upload via the S3Upload lib
  onChangeFiles: (e) ->
    return if $(e.target).val() == ''

    @uploadToS3
      files_dropped: false,
      file_dom_selector: '#Files'

  ...

  # Instantiation of a new S3Upload with custom callbacks
  uploadToS3: (options) ->
    # We create an object to store the newly created models for reference in progress and abort callbacks
    newAttachments = {}
    s3upload = new S3Upload
      # files_dropped and file_list are only necessary for handling drag and drop uploads, which we'll address later
      files_dropped: options.files_dropped,
      file_list: options.file_list,
      file_dom_selector: options.file_dom_selector,
      s3_sign_put_url: '/api/attachments/signput',

      onProgress: (xhr, file, percent, message) =>
        # Create a new Attachment model for the file which has just started uploading, otherwise trigger an 'upload:progress' event on the model which a view can listen for
        if percent == 0
          attachment = new Doozie.Models.Attachment
            name : file.name, 
            s3_url: '', 
            card_id : @model.id,
            project_id: @model.get('project_id'),
            file_type : file.type, 
            size : parseFloat(file.size)
          # Key on file.size for uniqueness
          newAttachments[file.size] = attachment
          # Add the model to the Backbone collection, which will trigger an 'add' event
          @attachmentsCollection.add(attachment)
        else
          attachment = newAttachments[file.size]
          if attachment
            attachment.trigger('upload:progress', { percent: percent, message: message, xhr: xhr })

      onAbort: (file, message) =>
        attachment = newAttachments[file.size]
        attachment.destroy()
        delete newAttachments[file.size]
        # Must replace old input type=file. for some reason, it doesn't clear the value
        @$('input#Files').replaceWith("<input id='files' type='file' name='files[]' multiple />")

      onFinishS3Put: (public_url, file) =>
        # Grab the attachment model and update it with the final url
        # Also note that the model has not yet been saved, so the backend would be unaware of a cancelled upload
        attachment = newAttachments[file.size]
        attachment.save({ s3_url: public_url })
        
      onError: (file, status) =>
        console.log('Upload error: ', status)
        attachment = newAttachments[file.size]
        attachment.destroy()
        delete newAttachments[file.size]
{% endcodeblock %}

The attachment view(s) instantiated above looks like this:
{% codeblock lang:coffeescript %}
  ...

  events: 
    'click .cancel' : 'onCancelUpload'

  initialize: () ->
    @listenTo(@model, 'upload:progress', @onUploadProgress)
    @listenTo(@model, 'destroy', @destroy)

  onUploadProgress: (data) ->
    @$('.progress .meter').css('width', "#{data.percent}%")

    if data.percent == 100
      # If the upload is complete, remove the progress bar and cancel button
      @$('.progress, .cancel').remove()
      @_xhr = null
    else if !@_xhr
      # Else if @_xhr is not yet defined, save a reference to it
      @_xhr = data.xhr

  onCancelUpload: (e) ->
    # Leverage the saved xhr reference to abort() the upload
    if @_xhr
      @_xhr.abort()
{% endcodeblock %}

On the Rails side of things, we're really only storing the S3 url and a few other details about the file that we may want to display. The most important piece here is the after_destroy callback to actually delete the S3 object when the model is destroyed. Note that this uses the 'aws-sdk' gem. 

{% codeblock lang:ruby %}
  class Attachment
    include Mongoid::Document
    include Mongoid::Timestamps

    field :name, type: String
    field :s3_url, type: String
    field :file_type, type: String
    field :size, type: Float

    belongs_to :creator, class_name: 'User', inverse_of: nil
    belongs_to :card, inverse_of: :blobs, touch: true

    after_destroy :delete_from_s3

    def delete_from_s3
      s3 = AWS::S3.new access_key_id: ENV['S3_ACCESS_KEY'], secret_access_key: ENV['S3_SECRET_KEY']
      key = s3_url.sub("https://s3.amazonaws.com/#{ENV['S3_BUCKET_NAME']}/", "")
      s3.buckets[ENV['S3_BUCKET_NAME']].objects[key].delete
    end

  end
{% endcodeblock %}

And that's all there is to it! We've built a simple and efficient solution that allows users to upload multiple files at the same time, see indiviual progress bars for each file and also cancel each file independently. To take this a little further we can allow users to drag files onto the view and trigger an upload, and also create a proxy route for downloading files, which hides the actual S3 url. 

To upload files on 'drop', just add the following event listener and handler, and it will trigger the same sequence of events as with the file type input.
{% codeblock lang:coffeescript %}
  events: 
    'drop'  :  'onDrop'

  onDrop: (e) ->
    return if e.dataTransfer == null

    @uploadToS3({
      files_dropped: true,
      file_list: e.dataTransfer.files
    });
{% endcodeblock %}

For the download link, we just use a controller action mapped to /attachments/download/:id
{% codeblock lang:ruby %}
    def download
      @attachment = Attachment.where( company_id: current_user.company_id, id: params[:id] ).first
      data = open URI.parse(@attachment.s3_url)
      send_data data.read, filename: "#{@attachment.name}", type: "#{@attachment.file_type}", disposition: 'inline', stream: 'true', buffer_size: '4096'
    end
{% endcodeblock %}

