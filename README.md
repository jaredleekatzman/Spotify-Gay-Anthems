Spotify-Gay-Anthems
---

Data Science project to explore the most popular songs on Spotify playlists with the keyword 'gay'.

Code shown below and provided in iPython Notebook:

- Spotify Gay-Playlist Datascrape.ipynb
- billboard_top100.html : HTML file containing the Billboard Top 100 Songs from September 2015

Output of script on the following keywords are in .csv files:

- gay : spotify_gay_songs.csv
- lesbian : spotify_lesbian_songs.csv
- transgender : spotify_transgender_songs.csv
- bisexual : spotify_bisexual_songs.csv

iPython Notebook
---

```python
import json
import requests
import spotipy
import spotipy.util as util
import pprint
import pandas as pd
import numpy as np

from bs4 import BeautifulSoup
import re
from unidecode import unidecode

import os, io
import sys
```


```python
# Get Authorization from Spotify
# Replace with your own API keys
my_client_id = 'MY_CLIENT_ID'
my_secret = 'MY_SECRET_KEY'
my_redirect_uri = 'wgss200://callback'

scope = ''
spotify = spotipy.Spotify()

username = 'SPOTIFY_ACCOUNT_EMAIL_ADDRESS'

# PROMPT_FOR_USER_TOKEN should open up a new tab where you can sign in to Spotify
# if sign in is successful, it will redirect you to a new page that contains a token
# Copy and paste the URL into the input field
# Google Chrome has known to have issues wit this process. (I open the link and Firefox)
token = util.prompt_for_user_token(username,scope,my_client_id,my_secret,redirect_uri=my_redirect_uri)

if token:
    sp = spotipy.Spotify(auth=token)
    print 'Successfully received auth token'
else:
    print 'Cannot get token for', username
```

# Outline of Data Scraping Procedure
- For a given list of keywords, return a dataframe of songs from the playlists results for an individual keyword search
    - For each keyword:
        - Get a list of all the playlists
        - For each playlist:
            - Extract the tracks for each playlist
            - Append playlist information to song entry
        - Appends keyword data to each song entry
    - Combine list of tracks to one big dataframe


```python
def extract_track_data(track_dict):
    """
    EXTRACT_TRACK_DATA(track_dict) takes a 'track'-object and extracts the track's:
        track_id, track_name, track_popularity, track_artists 

    If track_dict is empty, return an empty dictionary
    """
    track = {}
    if not track_dict:
        return track

    track['track_id'] = track_dict['id']
    track['track_name'] = track_dict['name']
    track['popularity'] = track_dict['popularity']
    track['artists'] = [artist['name'] for artist in track_dict['artists']]

    return track

def get_tracks(playlist, keyword):
    """
    Takes a Spotify playlist dictionary-object and returns
    a list of entry dictionaries with the relevant track and playlist info
    """
    if not playlist:
        return []
    
    playlist_user = playlist['owner']['id']
    playlist_id = playlist['id']
    
    # Results are returned as 'pages' (a set with a fixed number of songs)
    result = sp.user_playlist_tracks(playlist_user, playlist_id)
    
    # Handle 1 page playlists
    if result.has_key('items'):
        tracks = result['items']
    else:
        return []
    
    # The method sp.next(result) returns the next page after
    # extracting the tracks of that page
    while result['next']:
        result = sp.next(result)
        tracks.extend(result['items'])
    
    # Now we need to extract the relevant information
    entries = []
    
    # Takes the track dictionary-object and adds playlist metadata:
    #      playlist_id, playlist_name, playlist_user
    for track in tracks:
        entry = extract_track_data(track['track'])
        entry['playlist_id'] = playlist_id
        entry['playlist_owner'] = playlist_user
        entry['playlist_name'] = playlist['name']
        entry['keyword'] = keyword
        
        entries.append(entry)
        
        print '[INFO] Appended track: ', entry
        
    return entries
        
def get_playlists(sp, keyword):
    """
    GET_PLAYLISTS accepts a search query KEYWORD and returns 
    a list of playlist dictionary-objects
    """
    results = sp.search(q=keyword,type='playlist')
    
    # A list of dictionarys; each dictionary is a 'playlist' object
    playlists = results['playlists']['items']

    # Results are also page objects
    # sp.next gets the next page of playlists
    while results['playlists']['next']:
        results = sp.next(results['playlists'])
        playlists.extend(results['playlists']['items'])

    return playlists
    
def get_songs_for_keywords(sp, keywords, save = True):
    """
    Returns a pandas.DataFrame where each row is a track found 
    in a playlist by searching Spotify for each keyword in KEYWORDS
    
    If SAVE == True: append tracks to a csv file for each keyword
        Filename: spotify_{$KEYWORD}_songs.csv
    """
    song_df = pd.DataFrame()
    for keyword in keywords:
        song_csv_file = 'spotify_%s_songs.csv' % keyword

        playlists = get_playlists(sp, keyword)
        
        index = 0 
        for playlist in playlists:
            print '[INFO] Playlist Number: ',index
            try:
                track_entries = get_tracks(playlist, keyword)
            except:
                track_entries = []
                print 'Error extracting tracks'
            
            tracks_df = pd.DataFrame(track_entries)
            if save:
                csv_string = tracks_df.to_csv(header=True,index=False)
                with open(song_csv_file,'ab') as output:
                    output.write(csv_string)
                
            song_df = song_df.append(tracks_df, ignore_index=True)
            index = index + 1
        
    return song_df        
```

# Research:
Extract playlists for the following keywords:
    - gay
    - lesbian
    - bisexual
    - transgender


```python
reload(sys)  
sys.setdefaultencoding('utf8')

spotify_gay_dataset = get_songs_for_keywords(['gay'], save = True)
spotify_LBTQ = get_songs_for_keywords(['lesbian','bisexual','transgender'], save = True)
```

# Results
Only the gay-keyword dataset had a substantial amount of entries. 

We will look for the highest occuring songs in the dataset


```python
# Loading dataset from saved .csv file 
gdf = pd.read_csv('spotify_gay_songs.csv')
gdf.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>artists</th>
      <th>keyword</th>
      <th>playlist_id</th>
      <th>playlist_name</th>
      <th>playlist_owner</th>
      <th>popularity</th>
      <th>track_id</th>
      <th>track_name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[u'Ke$ha']</td>
      <td>gay</td>
      <td>3unWhWfWvs8UGxLGZoaFYF</td>
      <td>GAY ANTHEMS</td>
      <td>1190853721</td>
      <td>78</td>
      <td>6mnjcTmK8TewHfyOp3fC9C</td>
      <td>Die Young</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[u'Britney Spears']</td>
      <td>gay</td>
      <td>3unWhWfWvs8UGxLGZoaFYF</td>
      <td>GAY ANTHEMS</td>
      <td>1190853721</td>
      <td>71</td>
      <td>717TY4sfgKQm4kFbYQIzgo</td>
      <td>Toxic</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[u'Nicki Minaj']</td>
      <td>gay</td>
      <td>3unWhWfWvs8UGxLGZoaFYF</td>
      <td>GAY ANTHEMS</td>
      <td>1190853721</td>
      <td>73</td>
      <td>2EBCVPNAG46nbgs6jXPGvv</td>
      <td>Starships</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[u'Spice Girls']</td>
      <td>gay</td>
      <td>3unWhWfWvs8UGxLGZoaFYF</td>
      <td>GAY ANTHEMS</td>
      <td>1190853721</td>
      <td>79</td>
      <td>1Je1IMUlBXcx1Fz0WE7oPT</td>
      <td>Wannabe - Radio Edit</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[u'Katy Perry', u'Juicy J']</td>
      <td>gay</td>
      <td>3unWhWfWvs8UGxLGZoaFYF</td>
      <td>GAY ANTHEMS</td>
      <td>1190853721</td>
      <td>81</td>
      <td>4kgsK0fftHtg9gZOzkU5T2</td>
      <td>Dark Horse</td>
    </tr>
  </tbody>
</table>
</div>




```python
from collections import Counter

song_id2name = {song_id : song_name 
                for (song_id, song_name) in zip(gdf['track_id'],gdf['track_name'])}
counter = Counter(gdf['track_id'])

# NaN will be the highest count - remove it
song_id = counter.most_common(11)[1:]

top_10 = [(song_id2name[i], count) for (i, count) in song_id]
print top_10
```

    [('Where Are \xc3\x9c Now (with Justin Bieber)', 127), ('See You Again (feat. Charlie Puth)', 109), ('Trap Queen', 102), ('Thinking Out Loud', 94), ('Stole the Show', 87), ('Firestone', 84), ('Chandelier', 79), ('Shut up and Dance', 76), ('Want To Want Me', 76), ('Worth It', 76)]


These results were a little too Top 100. I want to see if I can find a more distinct list of gay-anthems, so I filtered out any song on the Top 100


```python
with open('billboard_top100.html') as fp:
    soup = BeautifulSoup(fp.read(), 'html')
```


```python
# Each Song is in an <article> tag with a tag ID="row-#"
# and data-hovertracklabel = 'Song Hover-{$SONGNAME}'
charts = soup.find_all('article', id=lambda x: x.startswith('row-') if x else False)
charts = [re.sub('Song Hover-', '', label['data-hovertracklabel']) for label in charts]
print charts
assert len(charts) == 100
```

    ['What Do You Mean?', "Can't Feel My Face", 'The Hills', 'Watch Me', 'Cheerleader', 'Lean On', 'Good For You', '679', 'Locked Away', 'Where Are U Now', 'Cool For The Summer', 'Photograph', 'Fight Song', 'Trap Queen', 'Wildest Dreams', 'My Way', 'Shut Up And Dance', 'Downtown', 'Stitches', 'See You Again', 'Bad Blood', 'Hotline Bling', 'Drag Me Down', 'Hit The Quan', 'Uptown Funk!', 'Marvin Gaye', 'Uma Thurman', 'All Eyes On You', 'Worth It', 'Classic Man', 'Flex (Ooh Ooh Ooh)', 'Want To Want Me', 'House Party', 'Thinking Out Loud', "Honey, I'm Good.", 'Sugar', 'Earned It (Fifty Shades Of Grey)', 'Post To Be', 'Back To Back', 'Renegades', 'John Cougar, John Deere, John 3:16', 'Again', 'Hey Mama', 'Love Myself', 'Buy Me A Boat', 'Crash And Burn', 'Prisoner', 'Strip It Down', 'Planes', "Ex's & Oh's", "Should've Been Us", 'Where Ya At', "Like I'm Gonna Lose You", 'Tell Your Friends', 'How Deep Is Your Love', 'This Could Be Us', "I Don't Like It, I Love It", "She's Kinda Hot", 'Lose My Mind', 'Acquainted', 'Hell Of A Night', 'Real Life', 'Save It For A Rainy Day', 'Kick The Dust Up', 'El Perdon', 'Here', 'Burning House', 'Comfortable', 'Fly', 'Levels', 'Beautiful Now', 'Ghost Town', 'Anything Goes', 'Black Magic', 'Break Up With Him', 'Smoke Break', 'Roots', "I'm Comin' Over", 'Shameless', 'Rotten To The Core', 'Cheyenne', 'One Man Can Change The World', 'Loving You Easy', 'Alright', 'Losers', 'Do It Again', 'R.I.C.O.', 'Let Me See Ya Girl', 'Omen', 'Young & Crazy', '100', 'No Role Modelz', 'Dark Times', "Nothin' Like You", 'Jet Black Heart', 'Gonna Wanna Tonight', 'New Americana', 'The Night Is Still Young', 'Real Life', 'In The Night']



```python
# Get most common 105 songs - in case all songs are in the top 100 (highly unlikely)
song_id = counter.most_common(106)[1:]
top_105 = [(song_id2name[i], count) for (i, count) in song_id]

def is_top_billboard(song, billboards):
    # Unidecode converts unicode characters to their closest ASCII equivalent
    song = unidecode(song)
    
    # Billboard song names are stripped names
    # Spotify song names have extra details (e.g. ft. ARTIST, '- Radio Edit')
    for s in billboards:
        if re.search(s, song, re.IGNORECASE):
            return True
    return False
```


```python
top_gay_songs = [(song,count) for (song,count) in top_105 if not is_top_billboard(unicode(song,'utf-8'),charts)]
print top_gay_songs[:5]
```
Now these are songs that make more sense:

    [('Stole the Show', 87), ('Firestone', 84), ('Chandelier', 79), ('Wannabe - Radio Edit', 72), ('Believe', 67)]

