import sys
import os
from flask import (
    Flask, flash, render_template, request, redirect,
    session, url_for, send_from_directory, jsonify
)
from pathlib import Path
from werkzeug.utils import secure_filename
from flask_wtf.csrf import CSRFProtect

# Ensure BASE_DIR is set correctly
BASE_DIR = os.environ.get("BASE_DIR", "C:\\Users\\saisi\\Downloads\\cmate-virtual-tryon-master\\cmate-virtual-tryon-master")
sys.path.append(BASE_DIR)

# Initialize the Flask application
def init_app():
    application = Flask(__name__)

    # Set up a secret key for CSRF protection
    application.config['SECRET_KEY'] = os.urandom(32)

    # Additional configurations if needed

    return application

# Create the Flask application instance
application = init_app()

# Set up CSRF protection
csrf = CSRFProtect(application)

# Define directories from application config (adjust these paths as per your actual structure)
PROFILE_IMAGES_DIR = os.path.join(BASE_DIR, 'src', 'flask_app', 'static', 'images', 'samples', 'profile_images')
SOURCE_IMAGES_DIR = os.path.join(BASE_DIR, 'src', 'flask_app', 'static', 'images', 'samples', 'source_images')
OUTPUT_IMAGES_DIR = os.path.join(BASE_DIR, 'src', 'flask_app', 'static', 'images', 'results')

# Import forms and functions after updating the sys.path
from flask_app.controller.image_upload_form import ProfileImageUploadForm, SourceImageUploadForm
from flask_app.controller.utils import session_active, fetch_image
from flask_app.controller.cmate import merge_images
from flask_app.controller.local_cache import cache_storage, update_cache, clear_cache

# Routes and application logic

@application.route('/')
def index():
    user_profile_image = None
    if session_active(PROFILE_IMAGES_DIR):
        user_profile_image = session['profile_image']
    return render_template('index.html', profile_image=user_profile_image)

@application.route('/footer/<path:link>')
def footer_page(link):
    try:
        return render_template('footer/' + link)
    except Exception:
        return render_template('404.html'), 404

@application.errorhandler(404)
def not_found_error(e):
    return render_template('404.html'), 404

@application.route('/upload_profile_image/', methods=['GET', 'POST'])
def upload_profile():
    if session_active(PROFILE_IMAGES_DIR) and not request.args.get('updateprofile'):
        return redirect(url_for('upload_source'))

    profile_form = ProfileImageUploadForm()
    if request.method == "POST":
        if profile_form.validate_on_submit():
            file = profile_form.profile_image.data
            filename = secure_filename(file.filename)
            file.save(str(Path(PROFILE_IMAGES_DIR) / filename))
            session['profile_image'] = filename

            if request.args.get('nextpage'):
                return redirect(url_for(request.args.get('nextpage')))

            return redirect(url_for('upload_source'))

    user_profile_image = None
    if session_active(PROFILE_IMAGES_DIR):
        user_profile_image = session['profile_image']
    return render_template('upload_profile_image.html', form=profile_form, profile_image=user_profile_image)

@application.route('/upload_source_image/', methods=['GET', 'POST'])
def upload_source():
    source_form = SourceImageUploadForm()
    if request.method == "POST":
        if source_form.validate_on_submit():
            image_url = source_form.source_image_url.data
            application.logger.info("Downloading image: " + image_url)
            source_image = fetch_image(image_url, dest_folder=SOURCE_IMAGES_DIR)
            if "404" in source_image:
                flash(source_image, 'danger')
                return redirect(url_for('upload_source'))

            session['source_image'] = source_image
            return redirect(url_for('tryon'))

    uploaded_source_image = None
    if 'source_image' in session:
        if (Path(SOURCE_IMAGES_DIR) / session['source_image']).exists():
            uploaded_source_image = session['source_image']
    return render_template('upload_source_image.html', form=source_form, source_image=uploaded_source_image)

@application.route('/tryon/', methods=['GET', 'POST'])
def tryon():
    user_profile_image = "None"
    user_source_image = "None"
    tryon_result_image = "None"
    try:
        user_profile_image = session['profile_image']
        user_source_image = session['source_image']
        issues = ""

        if user_profile_image + user_source_image in cache_storage:
            tryon_result_image, issues = cache_storage[user_profile_image + user_source_image].split('--')

        if issues:
            flash(issues, 'warning')

    except Exception as e:
        flash(str(e), 'danger')

    return render_template('tryon_result_page.html',
                           profile_image=user_profile_image,
                           source_image=user_source_image,
                           result_image=tryon_result_image)

@application.route('/tryon_result/', methods=['GET', 'POST'])
def tryon_result():
    try:
        user_profile_image = session['profile_image']
        user_source_image = session['source_image']

        result_image, issues = merge_images(user_profile_image, user_source_image,
                                            profile_dir=PROFILE_IMAGES_DIR,
                                            source_dir=SOURCE_IMAGES_DIR,
                                            dest_dir=OUTPUT_IMAGES_DIR)

        update_cache(user_profile_image + user_source_image, result_image + '--' + '\n'.join(issues))

        if issues:
            flash('\n'.join(issues), 'warning')

    except Exception as e:
        flash(str(e), 'danger')
        result_image = ""

    return jsonify(result_image=result_image)

@application.route('/download/<path:filename>')
def download_image(filename):
    return send_from_directory(OUTPUT_IMAGES_DIR, secure_filename(filename), as_attachment=True)

@application.route('/profile_images/<path:filename>')
def serve_profile_image(filename):
    return send_from_directory(PROFILE_IMAGES_DIR, secure_filename(filename))

@application.route('/source_images/<path:filename>')
def serve_source_image(filename):
    return send_from_directory(SOURCE_IMAGES_DIR, secure_filename(filename))

@application.route('/result_images/<path:filename>')
def serve_result_image(filename):
    return send_from_directory(OUTPUT_IMAGES_DIR, secure_filename(filename))

@application.route('/sample_profile_images/', methods=['GET', 'POST'])
def sample_profiles():
    sample_profiles = [str(path.name) for path in (Path(application.static_folder) / 'images' / 'samples' / 'profile_images').iterdir()]
    return jsonify(sample_profile_images=sample_profiles[:5])

@application.route('/sample_source_images/', methods=['GET', 'POST'])
def sample_sources():
    sample_sources = [str(path.name) for path in (Path(application.static_folder) / 'images' / 'samples' / 'source_images').iterdir()]
    return jsonify(sample_source_images=sample_sources[:5])

@application.route('/pick_profile_image/<path:filename>', methods=['GET', 'POST'])
def choose_profile_image(filename):
    filename = secure_filename(filename)
    session['profile_image'] = filename
    return redirect(url_for('upload_source'))

@application.route('/pick_source_image/<path:filename>', methods=['GET', 'POST'])
def choose_source_image(filename):
    filename = secure_filename(filename)
    session['source_image'] = filename
    return redirect(url_for('tryon'))

if __name__ == '__main__':
    application.run(debug=True)

# virtual-dressing-room
