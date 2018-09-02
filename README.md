django-easycomments

django-easycomments is a Django application which allows for the simple creation of a threaded commenting system. Commenters can reply both to the original item, and reply to other comments as well.

The application can be easily extended by other modules.

_Installation_

Install the package via pip:
pip install django-easycomments
It's preferred to install the module in a virtual environment.

_Configuration_

Add the following to settings.py:

INSTALLED_APPS += (
    'django-easycomments'
)

COMMENTS_APP = 'django-easycomments'

----------------------------------------------------------------

Models.py:

Add this properties at the end of your model that you want to comment.
  from django.contrib.contenttypes.models import ContentType
  
  @property
    def comments(self):
        instance = self
        qs = Comment.objects.filter_by_instance(instance)
        return qs
        
  @property
  def get_content_type(self):
      instance = self
      content_type = ContentType.objects.get_for_model(instance.__class__)
      return content_type
      
----------------------------------------------------------------

Views.py:
  Integrate this codes in your (e.g. post_detail) view to add your model comment form and threads.
  instance is the model that you want to comment (e.g. post)
  
  from comments.models import Comment
  from comments.forms import CommentForm
 
  initial_data = {
            "content_type": instance.get_content_type, 
            "object_id": instance.id
    }
    
  form = CommentForm(request.POST or None, initial=initial_data)
  if form.is_valid():
      c_type = form.cleaned_data.get("content_type")
      content_type = ContentType.objects.get(model=c_type)
      obj_id = form.cleaned_data.get('object_id')
      content_data = form.cleaned_data.get("content")
      parent_obj = None
      try:
          parent_id = int(request.POST.get("parent_id"))
      except:
          parent_id = None

      if parent_id:
          parent_qs = Comment.objects.filter(id=parent_id)
          if parent_qs.exists() and parent_qs.count() == 1 :
              parent_obj = parent_qs.first()

      new_comment, created = Comment.objects.get_or_create(
                          user = request.user,
                          content_type= content_type,
                          object_id = obj_id,
                          content = content_data,
                          parent = parent_obj,
                          )
      return HttpResponseRedirect(new_comment.content_object.get_absolute_url()) 
      
      
  comments = Comment.objects.filter_by_instance(instance)   
  context={
        "instance" : instance,
        "comments" : comments,
        "comment_form":form,
        }
   return render(request, "app_name/post_detail.html",context)

----------------------------------------------------------------

Provide a template (e.g. post_detail.html) that displays the comments for the object (e.g. article or blog entry):

Comment form:

  {% if request.user.is_authenticated %}
      <p>Comment</p>
      <form method="POST"> {% csrf_token %}
        {{ comment_form|crispy }}
        <input type='submit' value='Comment' class='btn btn-outline-success'>
      </form>
  {% endif %}

  {% for comment in comments %}
  
        {{ comment.content }}
        <footer class="blockquote-footer">{{ comment.user }} | {{ comment.timestamp|timesince }} ago | {% if comment.children.count > 0 %}{{ comment.children.count }} Comment{% if comment.children.count > 1 %}s{% endif %} | {% endif %} <a class='comment-reply-btn' href='#'> Reply </a> | <a href='{{ comment.get_absolute_url }}'>Thread</a></footer>

        {% for child_comment in comment.children %}
              {{ child_comment.content }}
        {% endfor %}

        {% if request.user.is_authenticated %}
          <form method="POST"> {% csrf_token %}
            {{ comment_form|crispy }}
            <input type='hidden' name='parent_id' value='{{ comment.id }}'>
            <input type='submit' value='Reply' class='btn btn-outline-dark'>
          </form>
        {% else %}
          <p>You must login to comment </p>
        {% endif %}

        {% empty %}
          <p >No comments posted yet.</p>

{% endfor %}
