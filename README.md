PROBLEM STATEMENT
	Spotify's 'daylists' are a fantastic way to enjoy unique blends of musical genres and moods. However, sometimes the 
	genre/mood labels of certain music (specifically more niche genres) are 'wrong' to different people. An
	example would be that I could feel the band 'Twenty One Pilots' is a pop-rock band, but my friend might see
	it as an 'emo' band. This application would allow users to change the labels of their favorite bands,
	specific albums or songs. It then would create new 'daylists' in the style of Spotify based on either their
	most listened to genres, or user inputted labels, first from the website's database, and then from the actual
	API itself. The website would also have a page called 'wacklists' that make funny versions of the daylists 
	just for a fun little side-feature. 

CORE FEATURES (MVP) 
	1. Creates genre/mood specific playlists with the time of day in the name (similar to 'daylists') based on user's 
	listening history.
	2. Mood and genre system
		MOODS
		- upbeat, chill, emotional, aggressive, melancholy, energetic, romantic, focus
		- Last.fm tags -> normalization lookup/mapping layer -> fixed list for users
		GENRES 
		- start with a fixed seeded list 
		- users can submit new genres on a dedicated page 
		- before saving, fuzzy search checks for near-matches and shows suggestions 
		- User-submitted genres require admin approval before appearing for others
	3. Account creation to save data 
		ONBOARDING: ask user if they want to give access to their listening history 
			YES
				- pull up to 30 top artist, albums, and 15 top genres from Spotify (and last.fm) 
				- store in USER_ARTISTS, USER_ALBUMS
				- can also check their saved library when generating playlists 
				- add a manual "refresh my data" button rather than auto-syncing constantly 
			NO
				- user picks 5 genre/moods and 5 artists manually 
				- taste quiz (pick artists you like from these 20) 
				- playlists generated from last.fm top/lesser known track mix for chosen artists and genres 
	4. Allows users to change the labels of certain artists/albums/songs
	
	PLAYLIST GENERATION LOGIC 
		1. PULL USER'S LISTENING HISTORY or TASTE QUIZ ANSWERS 
		2. FOR EACH SONG/ARTIST CHECK THE USER LABELS 
		3. MATCH MOOD + GENRE TO USER LABELS FIRST, NORMALIZED LAST.FM LABELS SECOND 
		4. BUILD PLAYLIST, POST TO SPOTIFY 
		

STRETCH FEATURES
	Creates wacky playlists with funny names with the same format as above
		- This is a stretch feature because ultimately it is not the main feature/selling point of this application.
	Allow users to submit a playlist (or playlists) and blend that playlist's music with the created playlist the 
	site gives. 
	BPM/key based mood classification 

TECH STACK + WHY 
	Next.js REACT
		There are many benefits to Next.js over vanilla react or html. React's virtual DOM allows this app to run fast 
		on anything and everything as opposed to html requiring a re-render every new page. Next.js has server side rendering
		which allows the full site to be rendered as soon as the user opens the site. This is something that vanilla REACT
		does not have, and it also helps for SEO since search-engine crawlers can read the content rather than an emtpy
		HTML shell.
	PostgreSQL
		The data required for this application is very structured and can be broken down into many small tables and joined
		with complex queries for the required database calls. This is why a relational database is perfect for this application.
		PostgreSQL in particular is great for its amazing performance, industry-standard use, and high scalability.
	Tailwind CSS 
		Allows for cleaner code and for components to have their CSS baked in (reusability).
	DrizzleORM
		Cleaner than raw SQL for the Postgres database.
	Spotify Web API
		User library, listening history, playlist creation 
	Last.fm API
		genre/mood tag data since Spotify deprecated their genre and audio feature endpoints
	Fuse.js	
		fuzzy search for genre submission validation
	NextAuth.js	
		Spotify OAuth, handles login so we never store passwords
	Hosting: Vercel (app) + Neon (database)
	

DATA MODELS / SCHEMA 
The following relations were derived using 3NF synthesis: 

USERS
	user_id (PK)
	spotify_id
	display_name
	email
	created_at
	
ARTISTS
	artist_id (PK)
	spotify_id
	artist_name 

ALBUMS 
	album_id (PK)
	spotify_id
	album_title 
	artist_id (FK -> ARTISTS) 

SONGS 
	song_id (PK)
	spotify_id 
	song_title
	album_id (FK -> ALBUMS)
	artist_id (FK -> ARTISTS)
	
USER_LABELS
	label_id (PK)
	user_id (FK -> USERS)
	entity_type ("artist", "album", or "song")
	entity_id (the artist_id, album_id, or song_id)
	genre
	mood 
	
USER_ARTISTS 
	user_id (FK -> USERS)
	artist_id (FK -> ARTISTS) 
	
USER_ALBUMS
	user_id (FK -> USERS)
	album_id (FK -> ALBUMS)

PLAYLISTS
	playlist_id (PK)
	user_id (FK -> USERS)
	spotify_id
	playlist_name
	created_at 
	
GENRES 
	genre_id (PK)
	genre_name 
	approved (boolean, default false) 
	submitted_by (FK -> USERS) 
	

API / Route Plan

GET /api/genres 
	Description: Returns the available genres for relabeling
	
	Auth required: No
	
	Request body: None 
	
	Response (success 200): 
	{
		genres: [
			{
				genre_id: string, 
				genre_name: string
			}
		]
	}
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 

GET /api/users/me/user_labels  
	Description: Returns the LIU's custom labels 
	
	Auth required: Yes 
	
	Request body: None 
	
	Response (success 200): 
	{
		albums: [
			{
				entity_id: string, 
				genre: string, 
				mood: string
			}
		],
		songs: [
			{
				entity_id: string, 
				genre: string, 
				mood: string
			}
		],
		artists: [
			{
				entity_id: string, 
				genre: string, 
				mood: string
			}
		]
	}
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 

GET /api/users/me
	Description: Returns the current logged in user their user_id
	
	Auth required: Yes 
	
	Request body: None -- user identity comes from sesison cookie
	
	Response (success 200):
	{
		user_id: string,
		display_name: string, 
		email: string
	}
	
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 

POST /api/me/user_

POST /api/playlists/generate 
	Description: Generates a daylist-style playlist for the logged in user 
	
	Auth required: Yes 
	
	Request body: 
	{
		mood: string, // one of fixed vocabulary values
		genre: string,// genre_id from GENRES table 
		time_of_day: string // "early-morning" | "pre-lunch" | "afternoon" | "evening" | "night" | "late-night"
	}
	
	Response (success 200):
	{
		playlist_id: string,
		playlist_name: string, 
		spotify_id: string,
		tracks: [
			{
				song_id: string,
				song_title: string,
				spotify_id: string,
				artist_name: string
			}
		]
	}
	
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 

GET /api/users/me/user_albums 
	Description: Gets the LIU's saved albums
	
	Auth required: Yes
	
	Request body: none
	
	Response (success 200): 
	{
		album_id: string,
		artist_id: string, 
		spotify_id: string,
		tracks: [
			{
				song_id: string, 
				song_title: string, 
				spotify_id: string,
				artist_name: string 
			}
		]
	}
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 
	
GET /api/users/me/user_artists 
	Description: Gets the LIU's saved artists
	
	Auth required: Yes
	
	Request body: none
	
	Response (success 200):
	{
		artists: [
			{
				artist_id: string,
				artist_name: string,
				spotify_id: string
			}
		]
	}
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 
	
POST /api/users
	Description: Creates a user object and adds it to the user relation 
	
	Auth required: No 
	
	Request body: 
	{
		display_name: string, 
		email: string, 
		spotify_id: string,
		historyAccess: boolean		
	}

	Response (success 200):
	{
		user_id: string
	}
	
	Response (error 400): Please check your syntax and try again
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 
	
POST /api/users/me/user_artists 
	Description: Adds an artist to the LIU's account library 
	
	Auth required: Yes
	
	Request body: 
	{
		artist_id: string
	}
	
	Response (success 200): Artist added to library 
	Response (error 400): Artist does not exist
	Response (error 401): Unauthorized -- user not logged in 
	Response (error 500): Internal server error 

DELETE /api/users/me/user_artists/{artist_id} 
	Description: Removes the specified artist from LIU's account library. 
	
	Auth required: Yes 
	
	Request body: None 
	
	Response (success 200): Artist removed from library
	Response (error 400): Artist does not exist 
	Response (error 500): Internal server error

POST /api/users/me/user_albums 
	Description: Adds an album to the LIU's account library 
	
	Auth required: Yes
	
	Request body: 
	{
		album_id: string
	}
	
	Response (success 200): Album added to library 
	Response (error 400): Album does not exist
	Response (error 500): Internal server error 

POST /api/users/me/genres
	Description: Allows LIU to add a genre to the genre list 
	
	Auth required: Yes 
	
	Request body: 
	{
		genre_name: string
	}
	
	Response (success 200): Genre successfully added to the approval queue.
	Response (error 500): Internal server error 
	
PUT /api/user/me/user_labels/{entity_id}
	Description: Allows LIU to add a label of the specified entity 
	
	Auth required: Yes 
	
	Request body: 
	{
		entity_type: string, 
		genre: string,
		mood: string
	}
	
	Response (success 200): Label changed successfully 
	Response (error 400): Entity does not exist 
	Response (error 500): Internal server error 

DELETE /api/user/me/user_labels/{entity_id}
	Description: Allows LIU to remove a label of the specified entity 
	
	Auth required: Yes 
	
	Request body: none 
	
	Response (success 200): Label deleted successfully 
	Response (error 400): Label does not exist 
	Response (error 500): Internal server error 

DELETE /api/users/me/user_albums/{album_id} 
	Description: Removes the specified album from LIU's account library. 
	
	Auth required: Yes 
	
	Request body: None 
	
	Response (success 200): Album removed from library
	Response (error 400): Album does not exist 
	Response (error 500): Internal server error

SPOTIFY API ROUTES
	GET /albums/{id}
	GET /albums/{id}/tracks
	GET /me/albums 

	GET /artists/{id}
	GET /artists/{id}/albums 

	GET /tracks/{id}
	GET /me/tracks 

	GET /me/library/contains 

	GET /playlists/{playlist_id}
	GET /playlists/{playlist_id}/items 
	GET /me/playlists
	
	PUT /playlists/{playlist_id}/items
	POST /playlists/{playlist_id}/items
	POST /me/playlists
LAST.FM API ROUTES
	last.fm uses a method system via GET calls
	methods include:
	'Th(ese) service(s) do not require authentication.'
		album.getTags <- can match with album name from spotify
			album.getInfo <- in case matching doesnt work
		album.getToptags <- probably more reliable then getTags
		
		artist.getTopAlbums
		artist.getTopTags 
		artist.getTopTracks 
		artist.getSimilar
		
		tag.getSimilar <- GREAT AND SUPER HELPFUL FOR CREATING FUN UNIQUE playlists
		tag.getTopAlbums
		tag.getTopArtists
		tag.getTopTracks
		
		chart.getTopArtists <- great for "top 40 mix" 
		chart.getTopTracks
		
		track.getInfo 
		track.getTags 
		track.getTopTags 
		track.search 
		
Milestones
...
SUMMARIES

notes about last.fm api for me 
	methods that respond in REST style xml. (different from the usual POST GET etc)
	
	send a method param expressed as package.method along with method specific args
		- use identifiable user-agent header
		- be reasonable in usage and don't make excessive api calls
		- please use scrobbling to allow users to end listening data in to their last.fm user profiles 
	
	album.getInfo: 
	/2.0/?method=album.getinfo&api_key=YOUR_API_KEY&artist=Cher&album=Believe&format=json
