---
publish: true
layout: post
title: "Generating PDFs From Templates in Ruby on Rails and Heroku"
date: 2014-05-06 20:42
comments: true
keywords: rails, ruby on rails, heroku, pdf, pdf generation, pdftk, wickedpdf, wkhtmltopdf
description: A how-to on generating PDFs from a pre-assembled PDF template in a Ruby on Rails app hosted on Heroku. 
cover: 
---

With Ruby and more specifically Rails, there are many different gems for generating PDFs. In the past, I’ve had success with WickedPDF to convert HTML templates into PDF documents. However, it can be a royal pain to get the styling correct, and it’s even more tricky if you get into multi-page documents. One of the more popular alternatives to the wkhtmltopdf based solutions(like WickedPDF and PDFKit), is Prawn. Prawn is a pure Ruby PDF writing library that provides support for all sorts of text and graphic rendering options. If the PDF you’re looking to produce is comprised of a lot of dynamic data, like a receipt or invoice, this is probably the best route. 

<!--more-->

However, for one of our recent projects, I simply needed to populate a user name and two dates into a PDF template that was already designed and created with Adobe InDesign. So rather than recreate the whole thing with Prawn, I found a gem called pdf-forms that allows you to programmatically fill out PDF Form Fields. This gem saved me a ton of time, and even allows the business team to make quick design changes to the templates without any coding required. 

### Prequisites
[pdf-forms](https://github.com/jkraemer/pdf-forms) is a ruby wrapper for [PDFtk](http://www.pdflabs.com/tools/pdftk-server/). Installing on your local dev machine is done by downloading the appropriate installer from [here](http://www.pdflabs.com/tools/pdftk-server/). On OSX, this installs PDFtk at `/usr/local/bin/pdftk`. Heroku servers don’t have pdftk installed out of the box, so we'll need to package that into our app.

### Implementation
I found a blog post by [Adam Albrecht](http://adamalbrecht.com/2014/01/31/pre-filling-pdf-form-templates-in-ruby-on-rails-with-pdftk) that provided some great examples for representing each PDF as a class. I took his same approach and added support for saving the PDFs to S3 and deploying the app to Heroku. So I recommend starting with Adam’s FillablePdfForm base class and extending from there. If you follow his example, you should be able to generate dynamic PDFs from the Rails console. For the app I was working on, we didn’t want to generate each PDF on-the-fly every time a user needed to access it. This should reduce the server load and also allow us to know exactly which file the user downloaded, should we need to debug issues in the future. 

To persist the PDF to S3, I chose to use the popular paperclip gem(I actually used mongoid-paperclip, since the app is using MongdoDB and mongoid) and aws-sdk. On the model, I added the following:
{% codeblock lang:ruby %}
  has_mongoid_attached_file :provider_card_pdf,
    path: ':id/:filename',
    storage: :s3,
    s3_credentials: {
      bucket: ENV['S3_BUCKET_NAME'],
      access_key_id: ENV['S3_ACCESS_KEY'],
      secret_access_key: ENV['S3_SECRET_KEY']
    }

  validates_attachment_content_type :provider_card_pdf, content_type: ['application/pdf']
{% endcodeblock %}

And also a method to perform the actual generation:
{% codeblock lang:ruby %}
  def generate_provider_card
    # passing ‘self’ so the form has access to user-related values
    pdf_path = ProviderCardPDFForm.new(self).export
    new_name = name.gsub(' ', '') + '.pdf'

    pdf = File.open(pdf_path)
    self.provider_card_pdf = pdf
    # Override the illegible temp filename with something more readable
    self.provider_card_pdf_file_name = new_name
    # Save to trigger the persistence to S3
    save
    # cleanup the temp file
    File.delete(pdf_path)
  end
{% endcodeblock %}

So now we’ve got a method that we can trigger with an `after_save` callback or a controller action, and the generated PDF gets stored in an S3 Bucket. Now let’s get it working on Heroku.

### Deploying to Heroku
When it comes to Heroku and binaries, you have a couple of options. 1. include the binary in your repo or 2. use a buildpack to load it during slug compilation. I opted for the first option in this case, but there is a [Heroku Buildpack for PDFtk](https://github.com/millie/heroku-buildpack-ruby-pdftk), I just couldn’t quite get it to work. Instead, the same author, [Millie](https://github.com/millie) has another repo for the PDFtk source, [https://github.com/millie/pdftk-source](https://github.com/millie/pdftk-source). Grab the zip from here and extract the contents into /vendor/pdftk.

{% codeblock lang:ruby %}
  # I cloned the repo into a sibling directory
  git clone https://github.com/millie/pdftk-source
  cd pdftk-source
  tar xzvf pdftk.tar.gz
  mkdir ../rails_project/vendor/pdftk
  cp -R bin../rails_project/vendor/pdftk/
  cp -R lib ../rails_project/vendor/pdftk/
{% endcodeblock %}

Next we need to set the environment variable we referenced earlier to the new path.
`heroku config:set PDFTK_PATH=/app/vendor/pdftk/bin/pdftk`

We should be all set! Add and commit the /vendor/pdftk directory and push the new code to Heroku. 

###References
Prawn - https://github.com/prawnpdf/prawn
Pre-Filling PDF Form Templates in Ruby-on-Rails with PDFtk - http://adamalbrecht.com/2014/01/31/pre-filling-pdf-form-templates-in-ruby-on-rails-with-pdftk/#fnref:1
pdf-forms gem - https://github.com/jkraemer/pdf-forms
